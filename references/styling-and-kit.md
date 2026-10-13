# Styling and the kit

Read when: styling anything, adding or changing a kit piece, choosing colors/spacing/type, icons, skeletons.
Styling library = Unistyles 3 unless the project's recorded decisions say otherwise. Check the
Unistyles docs for the installed version before using an API.

## Unistyles basics
* `const styles = StyleSheet.create((theme, rt) => ({ … }))` at the **bottom** of the component file.
* Tokens only. The theme lives in `src/config/styles/` and mirrors the project's design tokens (its style source of truth):
  semantic colors (`background`, `foreground`, `card`, `primary`, `secondary`, `muted`, `accent`,
  `destructive`, `border` … and their `*Foreground`), `theme.sizes.*` (spacing, radius, type scale),
  `theme.size(n)` (4-pt grid), `theme.shared.*` (e.g. `screenSection` gap), `theme.colorWithAlpha()`.
* Never in features: hex/`rgba()` literals, raw `fontSize`/`fontFamily`, raw spacing numbers
  (a 1-px hairline is the exception). The hi-fi needs a token that doesn't exist → report it; the
  theme is changed deliberately by whoever owns the design system, not inside a feature.
* Light **and** dark come from the same tokens; flip appearance live on a device, not only at launch.
  Returning to "system" appearance: set the current device theme first, then re-enable adaptive themes
  (they only follow scheme *changes*).

## Variants and token-typed props
Kit APIs look like shadcn's: `variant`, `size`, `color`, typed from the theme so a typo fails to compile.
```ts
// types/styles.ts
export type Color = keyof AppTheme['colors'];
export type Size = keyof AppTheme['sizes'];
export type FontWeight = keyof AppTheme['fonts'];
```
```tsx
// components/ui/button.tsx (trimmed)
export interface ButtonProps extends PressableProps {
  title: string;
  variant?: 'primary' | 'secondary' | 'ghost' | 'destructive';
  size?: 'sm' | 'md' | 'lg';
  isLoading?: boolean;
}

const Button = ({ title, variant = 'primary', size = 'md', isLoading, disabled, style, ...rest }: ButtonProps) => {
  const { theme } = useUnistyles();
  const isDisabled = disabled || isLoading;
  styles.useVariants({ variant, size, disabled: isDisabled });
  return (
    <Pressable
      style={[styles.container, style]}
      disabled={isDisabled}
      accessibilityState={{ disabled: isDisabled, busy: isLoading }}
      {...rest}
    >
      {isLoading ? (
        <ActivityIndicator color={theme.colors.primaryForeground} />
      ) : (
        <Text style={styles.label} weight="medium">{title}</Text>
      )}
    </Pressable>
  );
};

export default Button;

const styles = StyleSheet.create((theme) => ({
  container: {
    alignItems: 'center',
    borderRadius: theme.sizes.full,
    borderCurve: 'continuous',
    variants: {
      variant: {
        primary: { backgroundColor: theme.colors.primary },
        secondary: { backgroundColor: theme.colors.secondary },
        ghost: { backgroundColor: 'transparent' },
        destructive: { backgroundColor: theme.colors.destructive },
      },
      size: {
        sm: { paddingVertical: theme.sizes.xs, paddingHorizontal: theme.sizes.md },
        md: { paddingVertical: theme.sizes.sm, paddingHorizontal: theme.sizes.lg },
        lg: { paddingVertical: theme.sizes.md, paddingHorizontal: theme.sizes.xl },
      },
      disabled: { true: { opacity: 0.5 } },
    },
  },
  label: { variants: { variant: { primary: { color: theme.colors.primaryForeground } /* … */ } } },
}));
```
Call `styles.useVariants(...)` before the JSX; use `compoundVariants` for combinations (selected + today).

## Dynamic values
* Style arrays: `style={[styles.header, { paddingTop: insets.top }]}` - the only allowed inline object
  is a dynamic value inside an array. Static values always go in the stylesheet.
* Runtime values inside the sheet: `rt.insets.top`, `rt.screen.width`.
* Never spread styles (`{ ...styles.card, flex: 1 }`): Unistyles 3 loses the binding. Use arrays.
* Theme values for non-style props (icon color, `placeholderTextColor`, spinner color): `useUnistyles()`
  (a thin `useTheme()` wrapper is fine).
* Third-party components only re-style when wrapped with `withUnistyles` (a third-party `SafeAreaView`
  kept the light background after switching to dark). Theme/config changes need a JS restart.

## Kit layers
| Layer | Folder | Holds | Rules |
|---|---|---|---|
| primitive | `components/primitive/` | `pressable`, `text`, `text-input`, `image` (expo-image), `bottom-sheet` (TrueSheet), `dropdown-menu` (zeego) | wrap RN / a library **once**; named exports; behaviour (haptics, keyboard dismiss) lives here |
| ui | `components/ui/` | `Text`, `Button`, `Icon`, `IconButton`, `Badge`, `Card`, `Skeleton`, `Switch`… | variants, token-typed props, a11y; no business logic, no fetching |
| shared | `components/shared/` | `ScreenHeader`, `ScreenSection`, `ListItem`, `FloatingFooter`, `EmptyState`, `ErrorState`, `withLayout` | app-level compositions of ui pieces |
| feature | `components/<feature>/` | everything else | assemble kit pieces; make no visual decisions of their own |

* Features never import RN `Pressable`, `Image`, `Text` or `TextInput` directly - always the kit
  (seen in production: the primitive bypassed 15 times, losing haptics; don't).
* Kit on demand: tokens first, then a piece is born when the current slice needs it, built to the hi-fi,
  then reused. No catalog up front, no component for a future feature, no variant nobody uses; a dev-only
  `/kit` route is optional and excluded from release bundles. shadcn/ui and React Native Reusables are
  references for APIs and accessibility, not dependencies.
* Wrap, don't rebuild: sheets, menus, toasts, pickers come from libraries; native controls from
  `@expo/ui` when the project chose it (switch, slider, segmented control, date picker).
* Booleans for state or appearance are fine in the kit (`isLoading`, `selected`, `disabled`); slots are
  `ReactNode` props (`rightComponent`, `footer`, `prefix`, `suffix`).

## The Pressable primitive
Every press gets visual feedback, a light haptic and a default role:
```tsx
// components/primitive/pressable.tsx
export interface PressableProps extends RNPressableProps {
  haptic?: boolean;
}

export const Pressable = ({ onPress, haptic = true, accessibilityRole = 'button', style, ...props }: PressableProps) => {
  const handlePress = (event: GestureResponderEvent) => {
    if (haptic) Haptics.impactAsync(Haptics.ImpactFeedbackStyle.Light);
    onPress?.(event);
  };
  return (
    <RNPressable
      {...props}
      accessibilityRole={accessibilityRole}
      onPress={handlePress}
      style={(state) => [typeof style === 'function' ? style(state) : style, state.pressed && styles.pressed]}
    />
  );
};

const styles = StyleSheet.create({ pressed: { opacity: 0.7 } });
```
Key outcomes add a notification haptic in the feature hook (`Haptics.notificationAsync(Success)` after a
purchase or a finished session).

## Type, icons, polish
* Type = `Text` variants (`size`, `weight`, `color`); native headers keep the system font (a custom
  `fontFamily` blanked the large title on iOS 26).
* Icons through one `Icon` component over a typed registry
  (`ICONS = { … } as const satisfies Record<string, ComponentType<LucideProps>>`, `<Icon name="ChevronRight" />`)
  or SF Symbols via `expo-symbols` - the icon set is a project decision. Icon-only buttons use `IconButton`,
  whose `accessibilityLabel` prop is required.
* iOS polish: `borderCurve: 'continuous'` on rounded surfaces; `hitSlop` on small targets; glass effects
  behind an availability check with a plain fallback.
* Skeletons: the kit `Skeleton` + one standalone `<name>-skeleton.tsx` per loading component, same layout tokens.
