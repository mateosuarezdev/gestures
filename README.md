# Gestures

A lightweight, flexible gesture system for handling touch and mouse interactions in web applications. Built with performance and coordination in mind.

## Features

- 🎯 **Unified API** - Single API for both touch and mouse events
- 🏆 **Priority System** - Coordinate multiple gestures with priority-based arbitration
- 📏 **Threshold Detection** - Configure minimum movement before gesture activates
- 🎨 **Direction Control** - Lock gestures to horizontal, vertical, or all directions
- 🤏 **Multi-touch Support** - Built-in pinch (scale) and rotation tracking
- ⚡ **Performance** - Optimized for 60+fps with RAF integration pattern
- ⚛️ **React Hook** - Simple hook for React components
- 🎛️ **Custom Conditions** - Fine-grained control with `canStart` callback

## Installation

```bash
bun/pnpm/yarn add @mateosuarezdev/gestures
```

```bash
npm install @mateosuarezdev/gestures
```

## Basic Usage

### Vanilla JavaScript

```typescript
import { createGesture } from "@mateosuarezdev/gestures";

const element = document.getElementById("draggable");

const gesture = createGesture(element, {
  name: "drag",
  direction: "all",
  threshold: 10,

  onStart: (details) => {
    console.log("Gesture started", details);
  },

  onMove: (details) => {
    element.style.transform = `translate(${details.deltaX}px, ${details.deltaY}px)`;
  },

  onEnd: (details) => {
    console.log("Gesture ended", details);
  },
});

gesture.init();

// Cleanup when done
gesture.destroy();
```

### React Hook

```typescript
import { useGesture } from "@mateosuarezdev/gestures/react";
import { useRef } from "react";

function DraggableComponent() {
  const elementRef = useRef<HTMLDivElement>(null);

  useGesture(
    elementRef,
    {
      name: "drag",
      threshold: 10,
      onMove: (details) => {
        elementRef.current!.style.transform = `translate(${details.deltaX}px, ${details.deltaY}px)`;
      },
    },
    []
  ); // deps array

  return <div ref={elementRef}>Drag me!</div>;
}
```

## API Reference

### `createGesture(element, options)`

Creates a new gesture instance.

**Parameters:**

- `element: HTMLElement` - The element to attach the gesture to
- `options: GestureOptions` - Configuration object

**Returns:** `Gesture` instance with `init()` and `destroy()` methods

### GestureOptions

```typescript
type GestureOptions = {
  name?: string; // Identifier (default: "gesture-{id}")
  priority?: number; // Higher = wins conflicts (default: 0)
  threshold?: number; // Min pixels to move (default: 0)
  direction?: "x" | "y" | "all"; // Lock direction (default: 'all')
  minTouches?: number; // Min touch points (default: 1)
  maxTouches?: number; // Max touch points (default: Infinity)
  passive?: boolean; // Passive event listeners (default: true)
  canStart?: (details) => boolean; // Custom start condition
  onStart?: (details) => void; // Gesture begins
  onMove?: (details) => void; // Gesture updates
  onEnd?: (details) => void; // Gesture completes
};
```

### GestureDetails

The details object passed to callbacks:

```typescript
type GestureDetails = {
  isTracking: boolean; // Currently tracking
  startX: number; // Initial X position
  startY: number; // Initial Y position
  currentX: number; // Current X position
  currentY: number; // Current Y position
  deltaX: number; // Total X movement
  deltaY: number; // Total Y movement
  velocityX: number; // X velocity (px/ms)
  velocityY: number; // Y velocity (px/ms)
  touchCount: number; // Number of active touches
  scale: number; // Pinch scale (1.0 = no change)
  rotation: number; // Rotation in degrees
  type: string; // Gesture name
  event?: TouchEvent | MouseEvent; // Original event
};
```

## Advanced Usage

### Priority and Coordination

When multiple gestures overlap, priority determines which gesture wins:

```typescript
// Lower priority - loses to pan
const swipe = createGesture(element, {
  name: "swipe",
  priority: 0,
  threshold: 50,
  onMove: (details) => {
    /* ... */
  },
});

// Higher priority - wins over swipe
const pan = createGesture(element, {
  name: "pan",
  priority: 10,
  threshold: 10,
  onMove: (details) => {
    /* ... */
  },
});
```

### Direction Locking

Restrict gestures to specific axes:

```typescript
// Horizontal scrolling only
createGesture(element, {
  direction: "x",
  threshold: 20,
  onMove: (details) => {
    // deltaY will still be tracked, but threshold only checks deltaX
    element.scrollLeft -= details.deltaX;
  },
});
```

### Custom Start Conditions

Use `canStart` for fine-grained control:

```typescript
createGesture(element, {
  canStart: (details) => {
    // Only start if gesture begins on right half of screen
    return details.startX > window.innerWidth / 2;
  },
  onMove: (details) => {
    /* ... */
  },
});
```

### Multi-touch Gestures

Track pinch and rotation:

```typescript
createGesture(element, {
  minTouches: 2,
  maxTouches: 2,
  onMove: (details) => {
    // Scale and rotation are automatically tracked
    element.style.transform = `scale(${details.scale}) rotate(${details.rotation}deg)`;
  },
});
```

### Performance Pattern with RAF

For smooth animations, throttle DOM updates using requestAnimationFrame (adapts to device refresh rate - 60Hz, 90Hz, 120Hz, etc.):

```typescript
let rafId = null;
let latestDetails = null;

const gesture = createGesture(element, {
  threshold: 10,

  onMove: (details) => {
    // Store latest data
    latestDetails = details;

    // Skip if RAF already scheduled
    if (rafId) return;

    // Schedule update for next frame
    rafId = requestAnimationFrame(() => {
      element.style.transform = `translate(${latestDetails.deltaX}px, ${latestDetails.deltaY}px)`;
      rafId = null;
    });
  },

  onEnd: (details) => {
    // Cancel pending RAF
    if (rafId) {
      cancelAnimationFrame(rafId);
      rafId = null;
    }

    // Apply final position
    element.style.transform = `translate(${details.deltaX}px, ${details.deltaY}px)`;
  },
});
```

## Common Patterns

### Drag to Reorder

```typescript
createGesture(item, {
  name: "reorder",
  threshold: 5,
  passive: false, // Prevent scrolling

  onStart: () => {
    item.classList.add("dragging");
  },

  onMove: (details) => {
    item.style.transform = `translateY(${details.deltaY}px)`;
    updateDropZones(details.currentY);
  },

  onEnd: (details) => {
    item.classList.remove("dragging");
    item.style.transform = "";
    commitReorder();
  },
});
```

### Swipe to Dismiss

```typescript
createGesture(card, {
  name: "dismiss",
  direction: "x",
  threshold: 20,

  onMove: (details) => {
    const progress = Math.abs(details.deltaX) / 200;
    card.style.transform = `translateX(${details.deltaX}px)`;
    card.style.opacity = 1 - progress;
  },

  onEnd: (details) => {
    if (Math.abs(details.deltaX) > 150) {
      // Dismiss
      animateOut(card);
    } else {
      // Snap back
      card.style.transform = "";
      card.style.opacity = "1";
    }
  },
});
```

### Pull to Refresh

```typescript
createGesture(scrollContainer, {
  name: "refresh",
  direction: "y",
  threshold: 10,

  canStart: (details) => {
    // Only start if scrolled to top and pulling down
    return scrollContainer.scrollTop === 0 && details.deltaY > 0;
  },

  onMove: (details) => {
    const pull = Math.min(details.deltaY, 100);
    refreshIndicator.style.height = `${pull}px`;
  },

  onEnd: (details) => {
    if (details.deltaY > 80) {
      triggerRefresh();
    } else {
      refreshIndicator.style.height = "0";
    }
  },
});
```

## How It Works

### Gesture Controller

A singleton `GestureController` manages all gestures:

- **Registration**: Each gesture gets a unique ID
- **Priority**: Higher priority gestures can "capture" control
- **Coordination**: Only one gesture can be active at a time
- **Events**: Emits `gestures:capture` when a gesture takes control

### Threshold System

Gestures don't activate until movement exceeds the threshold:

1. User touches element
2. Controller registers the gesture attempt
3. As user moves, delta is calculated
4. When threshold is passed, gesture "captures" control
5. `onStart` is called
6. Subsequent moves call `onMove`

This prevents accidental activation and allows multiple gestures to coexist.

### Touch vs Mouse

Both touch and mouse events are handled transparently:

- Touch events: Uses center of all touch points
- Mouse events: Uses cursor position
- Multi-touch: Automatically calculates scale and rotation

## Best Practices

1. **Always cleanup**: Call `destroy()` or use the React hook with proper deps
2. **Use RAF for DOM updates**: Prevents layout thrashing
3. **Set appropriate thresholds**: Prevents conflicts with scrolling
4. **Use direction locking**: Makes gestures feel more natural
5. **Non-passive when needed**: Set `passive: false` only when preventing default is required
6. **Priority hierarchy**: Give higher priority to more specific gestures

## Troubleshooting

**Gesture not starting:**

- Check `canStart` callback if defined
- Verify element exists and is visible
- Check touch count limits (`minTouches`/`maxTouches`)
- Ensure another gesture hasn't captured control

**Conflicts with scrolling:**

- Increase `threshold` value
- Use `direction` to lock axis
- Check `passive` setting
- Verify priority values

**Performance issues:**

- Implement RAF pattern for DOM updates
- Avoid heavy computations in `onMove`
- Use CSS transforms instead of layout properties
- Consider debouncing expensive operations

## TypeScript Support

Full TypeScript definitions are included. All types are exported:

```typescript
import type {
  GestureOptions,
  GestureDetails,
  GestureDirection,
} from "@mateosuarezdev/gestures";
```

## Author

Created by Mateo Suarez ([@mateosuarezdev](https://github.com/mateosuarezdev))

## License

This package is licensed under the **Mateo Suarez Free Use License v1.0 (2025)**.

See the [LICENSE.md](./LICENSE.md) file for full details.

SPDX-License-Identifier: LicenseRef-MateoSuarez-FUL-1.0
