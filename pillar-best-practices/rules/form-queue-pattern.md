# form-queue-pattern

For tools that open forms or wizards requiring user submission, defer completion signaling until the user actually submits. When the agent may call multiple form-opening tools in one response, queue them and execute sequentially.

## The Problem

When an agent responds to "Route billing tickets to Finance, escalate unanswered after 4 hours, create a macro for refund requests," it calls three form-opening tools. Each tool's `execute` handler calls `navigate()`, but only the last navigation survives — the first two forms are never seen.

This happens because `execute` returns immediately and the SDK marks the tool as complete. The next tool fires before the user has interacted with the form.

## When to Use

Apply this pattern when:
- A tool opens a form/wizard that requires the user to click submit
- The agent may call multiple such tools in a single response
- The SDK cannot detect form completion automatically (no callback from your form to the SDK)

Do NOT apply this pattern to:
- Query tools that return data immediately
- Navigation tools that just change the page (no form)
- Trigger tools that perform an API call and return a result synchronously

## The Pattern: Single Form

For a single form-opening tool, defer completion to the form's submit handler:

```tsx
import { usePillarTool, usePillar } from '@pillar-ai/react';

function useTriggerTools() {
  const { completeTool } = usePillar();

  usePillarTool({
    name: 'open_create_trigger',
    type: 'trigger_tool',
    description: 'Open the trigger creation form pre-filled with conditions and actions.',
    execute: (data) => {
      navigate(`/triggers/new?prefill=${encodeURIComponent(JSON.stringify(data))}`);
      // Do NOT return here — completion is deferred to form submit
    },
  });

  // In your form component's submit handler:
  // completeTool('open_create_trigger', true, { id: newTrigger.id });
  //
  // In your form component's cancel handler:
  // completeTool('open_create_trigger', false);
}
```

Key points:
- `execute` opens the form but does NOT return a value or call `completeTool`
- The form's submit handler calls `completeTool(name, true, resultData)`
- The form's cancel/back handler calls `completeTool(name, false)`

## The Pattern: Sequential Form Queue

When multiple form tools may arrive in one response, add a queue so only one form is active at a time:

```tsx
// hooks/useFormQueue.ts
import { useRef, useCallback } from 'react';
import { usePillar } from '@pillar-ai/react';

interface QueuedForm {
  toolName: string;
  data: Record<string, unknown>;
  open: () => void;
}

export function useFormQueue() {
  const queue = useRef<QueuedForm[]>([]);
  const active = useRef<QueuedForm | null>(null);
  const { completeTool } = usePillar();

  const processNext = useCallback(() => {
    if (queue.current.length === 0) {
      active.current = null;
      return;
    }
    const next = queue.current.shift()!;
    active.current = next;
    next.open();
  }, []);

  const enqueue = useCallback((toolName: string, data: Record<string, unknown>, open: () => void) => {
    const item: QueuedForm = { toolName, data, open };
    if (!active.current) {
      active.current = item;
      item.open();
    } else {
      queue.current.push(item);
    }
  }, []);

  const onSubmit = useCallback((resultData?: Record<string, unknown>) => {
    if (!active.current) return;
    completeTool(active.current.toolName, true, resultData);
    processNext();
  }, [completeTool, processNext]);

  const onCancel = useCallback(() => {
    if (!active.current) return;
    completeTool(active.current.toolName, false);
    processNext();
  }, [completeTool, processNext]);

  return { enqueue, onSubmit, onCancel };
}
```

Wire it into your tools:

```tsx
function useTriggerTools() {
  const { enqueue } = useFormQueue();
  const navigate = useNavigate();

  usePillarTool({
    name: 'open_create_trigger',
    type: 'trigger_tool',
    description: 'Open the trigger creation form pre-filled with conditions and actions.',
    execute: (data) => {
      enqueue('open_create_trigger', data, () => {
        navigate(`/triggers/new?prefill=${encodeURIComponent(JSON.stringify(data))}`);
      });
    },
  });

  usePillarTool({
    name: 'open_create_macro',
    type: 'trigger_tool',
    description: 'Open the macro creation form pre-filled with actions.',
    execute: (data) => {
      enqueue('open_create_macro', data, () => {
        navigate(`/macros/new?prefill=${encodeURIComponent(JSON.stringify(data))}`);
      });
    },
  });
}
```

Wire form submit/cancel to the queue:

```tsx
function CreateTriggerPage() {
  const { onSubmit, onCancel } = useFormQueue();

  const handleSubmit = (formData) => {
    const trigger = saveTrigger(formData);
    onSubmit({ id: trigger.id, name: trigger.name });
  };

  return (
    <TriggerForm onSubmit={handleSubmit} onCancel={onCancel} />
  );
}
```

## Anti-Patterns

### Don't return immediately from form-opening tools

```tsx
// Bad: SDK marks tool complete before user interacts with the form
execute: (data) => {
  navigate('/triggers/new');
  return { navigatedTo: '/triggers/new', prefilled: data };
}
```

### Don't call completeTool inside execute

```tsx
// Bad: completion fires before the form is even visible
execute: (data) => {
  navigate('/triggers/new');
  completeTool('open_create_trigger', true);
}
```

### Don't fire multiple navigations without queuing

```tsx
// Bad: only the last navigation survives
execute: (data) => {
  navigate('/triggers/new');
  // Agent calls this 3 times — forms 1 and 2 are lost
}
```

## Checklist

Before shipping form-opening tools:

- [ ] `execute` does NOT return a value or call `completeTool`
- [ ] Form submit handler calls `completeTool(name, true, data)`
- [ ] Form cancel/close handler calls `completeTool(name, false)`
- [ ] If multiple form tools exist, they share a queue
- [ ] Queue dequeues on both submit and cancel (no deadlock)
- [ ] Duplicate submit clicks are idempotent (check if active item matches)
- [ ] Tool guidance mentions that the form requires user confirmation
