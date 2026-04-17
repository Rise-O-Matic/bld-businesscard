# Architectural Decision Records

## ADR-001: Hover + class-based dual flip mechanism
- **Date**: 2026-04-14
- **Status**: Accepted
- **Context**: Saved cards originally flipped on click via `.flipped` class toggle. User requested hover-to-flip, then a "Flip All" button. Hover is transient (CSS `:hover`), but Flip All needs persistent state (class toggle). These could conflict.
- **Decision**: Use CSS `:hover` for individual card flip AND `.flipped` class for Flip All. When both apply (hovering a flipped card), the card flips back to front temporarily via `rotateY(0deg)` override.
- **Consequences**: Clean UX — hover always shows the opposite face. No JS needed for individual card hover. Flip All toggles `.flipped` class on all cards.
