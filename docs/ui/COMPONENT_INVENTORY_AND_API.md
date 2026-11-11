# UI Component Inventory & API Reference
**Document ID:** `KW-UI-002`  
**Deliverable:** B (Component Inventory and API Reference)  
**Parent Document:** [README.md](file:///e:/Projects/All%20working%20projects/App%20projects%20working/KinetiWall%20Engine%20(working)/docs/ui/README.md)  
**Classification:** Frontend API Reference / Component Catalog  

---

## 1. Foundational UI Primitives (Tier 2)

### 1.1 `Button`
Versatile action trigger supporting semantic variants, visual states, and async loading feedback.

```typescript
export interface ButtonProps extends React.ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: 'primary' | 'secondary' | 'ghost' | 'danger' | 'neon';
  size?: 'sm' | 'md' | 'lg';
  isLoading?: boolean;
  leftIcon?: React.ReactNode;
  rightIcon?: React.ReactNode;
  children: React.ReactNode;
}
```

* **Variants:**
  - `primary`: Solid neon cyan background (`#00F0FF`) with dark text (`#0A0B10`).
  - `secondary`: Dark acrylic glass (`#1A1D2E`) with subtle 1px border (`#2A2E45`) and white text.
  - `ghost`: Transparent background with neon cyan text on hover.
  - `danger`: Magenta crimson (`#FF0055`) with high-contrast text for destructive actions.
  - `neon`: Translucent cyan backdrop with vibrant outer glow (`box-shadow: 0 0 12px rgba(0,240,255,0.4)`).
* **Keyboard Accessibility:** Native `<button>` element; triggers on `Enter` and `Space`; visible 2px cyan focus ring (`focus-visible:ring-2 focus-visible:ring-neon-cyan`).

---

### 1.2 `TextInput`
Single-line text entry field with integrated validation states, leading/trailing icons, and clear actions.

```typescript
export interface TextInputProps extends Omit<React.InputHTMLAttributes<HTMLInputElement>, 'size'> {
  label?: string;
  error?: string;
  helperText?: string;
  leftIcon?: React.ReactNode;
  onClear?: () => void;
  size?: 'sm' | 'md' | 'lg';
}
```

* **Styling:** Dark void background (`#0F111A`), 1px dark border (`#23283E`), rounded corners (`8px`).
* **Focus State:** Transitions border to neon cyan (`#00F0FF`) with soft ambient glow.
* **Error State:** Border shifts to magenta (`#FF0055`); renders accessible error message below input.

---

### 1.3 `Switch`
Toggle control for binary settings (e.g. "Pause on Battery", "Hardware Acceleration").

```typescript
export interface SwitchProps {
  id: string;
  label: string;
  description?: string;
  checked: boolean;
  onChange: (checked: boolean) => void;
  disabled?: boolean;
}
```

* **Accessibility:** Built upon native `<button role="switch" aria-checked={checked}>`; toggles via `Space` or `Enter`.
* **Visual Motion:** Track background animates between muted gray (`#23283E`) and vibrant neon cyan (`#00F0FF`) over $150\text{ ms}$.

---

### 1.4 `Slider`
Continuous or stepped range selector for volume, playback speed, and video property customization.

```typescript
export interface SliderProps {
  id: string;
  label: string;
  value: number;
  min: number;
  max: number;
  step?: number;
  unit?: string;
  onChange: (value: number) => void;
  disabled?: boolean;
}
```

* **Accessibility:** Native `<input type="range">` with explicit `aria-valuemin`, `aria-valuemax`, and `aria-valuenow`. Supports keyboard adjustments via `ArrowLeft` / `ArrowRight`.

---

### 1.5 `Badge`
Compact visual status indicator for engine state, wallpaper tags, and resolution badges.

```typescript
export interface BadgeProps {
  variant?: 'cyan' | 'magenta' | 'emerald' | 'amber' | 'muted';
  size?: 'sm' | 'md';
  children: React.ReactNode;
}
```

---

## 2. Composed Interface Components (Tier 3)

### 2.1 `SearchBar`
Integrated search input with search icon, instant clear button, and debounced callback.

```typescript
export interface SearchBarProps {
  placeholder?: string;
  value: string;
  onChange: (value: string) => void;
  debounceMs?: number;
}
```

---

### 2.2 `ConfirmationDialog`
Modal dialog for confirming destructive actions (e.g. deleting a wallpaper, resetting engine settings).

```typescript
export interface ConfirmationDialogProps {
  isOpen: boolean;
  title: string;
  message: string;
  confirmLabel?: string;
  cancelLabel?: string;
  isDestructive?: boolean;
  onConfirm: () => void;
  onCancel: () => void;
}
```

* **Accessibility:** Traps focus within modal container; restores focus to trigger element on exit; closes on `Escape` key or backdrop click.

---

## 3. Domain-Specific KinetiWall Components (Tier 4)

### 3.1 `WallpaperCard`
Primary visual representation of a live wallpaper package in the Library grid.

```typescript
export interface WallpaperCardProps {
  wallpaper: {
    id: string;
    title: string;
    author: string;
    wallpaperType: 'video' | 'web';
    thumbnailUrl: string;
    resolution: string;
    durationSeconds?: number;
  };
  isActiveOnDisplay?: string; // e.g. "Display 1"
  onApply: (displayId: string) => void;
  onCustomize: () => void;
  onDelete: () => void;
}
```

* **Visual Design:** Dark acrylic glass card with subtle border glow on hover (`hover:border-neon-cyan/60`). Displays thumbnail image with duration pill and active assignment badge.
* **Security Guardrail:** All text properties (`title`, `author`) are rendered strictly as escaped text nodes, preventing XSS injection from untrusted package manifests (`DEC-SEC-05`).

---

### 3.2 `MonitorSelector`
Interactive multi-monitor canvas visualizing connected displays with physical aspect ratios.

```typescript
export interface MonitorSelectorProps {
  monitors: Array<{
    id: string;
    displayName: string;
    isPrimary: boolean;
    resolution: string;
    assignedWallpaperTitle?: string;
  }>;
  selectedMonitorId: string;
  onSelectMonitor: (id: string) => void;
}
```

---

### 3.3 `PlaybackControlGroup`
Global controls for wallpaper playback: Play/Pause, Mute/Unmute, and Volume Slider.

```typescript
export interface PlaybackControlGroupProps {
  isPaused: boolean;
  isMuted: boolean;
  volume: number;
  onTogglePause: () => void;
  onToggleMute: () => void;
  onVolumeChange: (volume: number) => void;
}
```

