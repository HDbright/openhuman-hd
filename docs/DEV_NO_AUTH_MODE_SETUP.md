# 本地无认证模式 (DEV_NO_AUTH_MODE) 功能文档

## 功能概述

此功能允许在本地开发环境中运行应用，无需登录或配置 OAuth。适用于开发和测试阶段。

## 快速启用

在 `app/` 目录下创建 `.env.local` 文件并添加：

```env
VITE_DEV_NO_AUTH_MODE=true
```

## 修改文件列表

### 1. `app/src/utils/config.ts`

**新增配置项：**
```typescript
/** Dev only: disable authentication entirely for local development. */
export const DEV_NO_AUTH_MODE = import.meta.env.DEV
  ? import.meta.env.VITE_DEV_NO_AUTH_MODE === 'true'
  : false;
```

**位置：** 在 `DEV_JWT_TOKEN` 定义之后添加。

---

### 2. `app/src/components/ProtectedRoute.tsx`

**修改内容：**

1. 导入配置：
```typescript
import { DEV_NO_AUTH_MODE } from '../utils/config';
```

2. 在组件开头添加无认证模式处理：
```typescript
// No-auth mode: skip all authentication checks
if (DEV_NO_AUTH_MODE) {
  if (isBootstrapping) {
    return <RouteLoadingScreen />;
  }
  return <>{children}</>;
}
```

---

### 3. `app/src/components/PublicRoute.tsx`

**修改内容：**

1. 导入配置：
```typescript
import { DEV_NO_AUTH_MODE } from '../utils/config';
```

2. 在组件开头添加无认证模式处理：
```typescript
// No-auth mode: never redirect, always show children
if (DEV_NO_AUTH_MODE) {
  if (isBootstrapping) {
    return <RouteLoadingScreen />;
  }
  return <>{children}</>;
}
```

---

### 4. `app/src/providers/CoreStateProvider.tsx`

**修改内容：**

1. 导入配置：
```typescript
import { DEV_NO_AUTH_MODE } from '../utils/config';
```

2. 添加模拟认证状态函数（在 `toSignedOutSnapshot` 之后）：
```typescript
function toMockAuthenticatedSnapshot(snapshot: CoreAppSnapshot): CoreAppSnapshot {
  const mockUserId = 'dev-user-id-12345';
  const mockSessionToken = 'dev-session-token-abc123';
  return {
    ...snapshot,
    auth: {
      isAuthenticated: true,
      userId: mockUserId,
      user: { id: mockUserId, email: 'dev@example.com' },
      profileId: null,
    },
    sessionToken: mockSessionToken,
    currentUser: {
      id: mockUserId,
      email: 'dev@example.com',
      name: 'Local Dev User',
    } as any,
    onboardingCompleted: true,
    chatOnboardingCompleted: true,
    analyticsEnabled: false,
  };
}
```

3. 修改初始化状态：
```typescript
export default function CoreStateProvider({ children }: { children: ReactNode }) {
  const [state, setState] = useState<CoreState>(() => {
    const initialState = getCoreStateSnapshot();
    if (DEV_NO_AUTH_MODE) {
      return {
        ...initialState,
        snapshot: toMockAuthenticatedSnapshot(initialState.snapshot),
      };
    }
    return initialState;
  });
```

4. 修改 `memoryTokenRef` 初始化：
```typescript
const memoryTokenRef = useRef<string | null>(
  DEV_NO_AUTH_MODE ? 'dev-session-token-abc123' : state.snapshot.sessionToken
);
```

5. 修改 `refreshCore` 函数：
```typescript
const refreshCore = useCallback(async () => {
  const requestId = ++snapshotRequestIdRef.current;
  let snapshot: CoreAppSnapshot;
  
  // No-auth mode: use mock authenticated snapshot
  if (DEV_NO_AUTH_MODE) {
    const baseSnapshot = normalizeSnapshot(await fetchCoreAppSnapshot().catch(() => ({
      auth: { isAuthenticated: false, userId: null, user: null, profileId: null },
      sessionToken: null,
      currentUser: null,
      onboardingCompleted: false,
      chatOnboardingCompleted: false,
      analyticsEnabled: false,
      meetAutoOrchestratorHandoff: false,
      localState: { encryptionKey: null, onboardingTasks: null },
      runtime: { screenIntelligence: null, localAi: null, autocomplete: null, service: null },
    })));
    snapshot = toMockAuthenticatedSnapshot(baseSnapshot);
  } else {
    snapshot = normalizeSnapshot(await fetchCoreAppSnapshot());
  }
  // ... 其余代码保持不变
}
```

---

### 5. `app/src/App.tsx`

**修改内容：**

1. 导入配置：
```typescript
import { DEV_FORCE_ONBOARDING, DEV_NO_AUTH_MODE } from './utils/config';
```

2. 修改 `useEffect`：
```typescript
// Onboarding gate: while `onboarding_completed=false`, force any non-
// onboarding route back to `/onboarding`. Once completed, bounce the
// user off `/onboarding` so they don't get stuck on the stepper.
useEffect(() => {
  // No-auth mode: skip onboarding gates entirely
  if (DEV_NO_AUTH_MODE) return;
  
  if (isBootstrapping || !snapshot.sessionToken) return;
  if (onboardingPending && !onOnboardingRoute) {
    console.debug(
      `[onboarding-gate] redirecting ${location.pathname} -> /onboarding (onboarding incomplete)`
    );
    navigate('/onboarding', { replace: true });
  } else if (!onboardingPending && onOnboardingRoute) {
    console.debug(
      `[onboarding-gate] redirecting ${location.pathname} -> /home (onboarding complete)`
    );
    navigate('/home', { replace: true });
  }
}, [
  isBootstrapping,
  snapshot.sessionToken,
  onboardingPending,
  onOnboardingRoute,
  location.pathname,
  navigate,
]);
```

---

### 6. `app/.env.example`

**新增配置示例：**
```env
# [optional] Dev only: disable authentication entirely for local development
# This allows testing the app without logging in or setting up OAuth
VITE_DEV_NO_AUTH_MODE=false
```

---

### 7. `app/src/components/BootCheckGate/BootCheckGate.tsx`

**修改内容：**

1. 导入配置：
```typescript
import { DEV_NO_AUTH_MODE } from '../../utils/config';
```

2. 在组件开头添加无认证模式处理：
```typescript
export default function BootCheckGate({ children }: BootCheckGateProps) {
  const { t } = useT();
  const dispatch = useAppDispatch();
  const coreMode = useAppSelector(state => state.coreMode.mode);

  // Dev mode: skip boot check entirely
  if (DEV_NO_AUTH_MODE) {
    return <>{children}</>;
  }
  // ... 其余代码保持不变
}
```

---

## 使用步骤

### 拉取上游更新后应用此功能

1. **拉取更新：**
   ```bash
   git pull upstream main
   ```

2. **检查修改的文件是否存在：**
   - `app/src/utils/config.ts`
   - `app/src/components/ProtectedRoute.tsx`
   - `app/src/components/PublicRoute.tsx`
   - `app/src/providers/CoreStateProvider.tsx`
   - `app/src/App.tsx`
   - `app/.env.example`
   - `app/src/components/BootCheckGate/BootCheckGate.tsx`

3. **按照上述修改内容逐一应用变更**

4. **创建或更新 `app/.env.local`：**
   ```env
   VITE_DEV_NO_AUTH_MODE=true
   ```

5. **重启开发服务器**

## 注意事项

- 此功能仅在开发环境（`import.meta.env.DEV === true`）下生效
- 生产环境中此配置项会自动禁用
- 模拟用户信息：
  - 用户 ID: `dev-user-id-12345`
  - 邮箱: `dev@example.com`
  - 用户名: `Local Dev User`
