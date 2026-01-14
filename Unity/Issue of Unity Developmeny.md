

## IPointerEnter/Exit Flickering with Collider2D

**Problem:** When using `IPointerEnter` and `IPointerExit` with Collider2D objects, the events trigger Enter then immediately Exit even when the mouse remains inside the collider bounds.

**Root Cause:** 
- IPointer events are designed for UI Canvas elements, not scene objects
- Physics2DRaycaster can have conflicts with UI elements or overlapping colliders
- Event system priority issues cause flickering detection

**Solution:** Use `OnMouseEnter()` and `OnMouseExit()` instead
- These methods use Unity's built-in physics raycasting directly
- More stable and reliable for Collider2D scene objects
- No dependency on Event System or Physics2DRaycaster
- Simpler implementation with better performance

**Requirements for OnMouseEnter/Exit:**
- GameObject must have a Collider2D component
- Object should NOT be on "Ignore Raycast" layer
- Active Camera must be present in the scene

```csharp
void OnMouseEnter()
{
    // Mouse entered collider
}

void OnMouseExit()
{
    // Mouse exited collider
}
```

