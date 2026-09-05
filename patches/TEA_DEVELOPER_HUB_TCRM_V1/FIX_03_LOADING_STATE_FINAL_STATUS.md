# FIX 03 — Loading State Final Status

Status: READY

Root cause fixed in `/opt/TBA/src/api/developer-hub-service.api.ts`.

The axios interceptor referenced `adminKey` without declaring it, causing a ReferenceError before Developer Hub API requests were sent. This made React Query remain in a loading state and left the Repository Status card rendering skeletons indefinitely.

Applied fix:

```ts
const adminKey = getDeveloperHubSuperAdminKey();
```

Verified:

- Developer Hub status endpoint returns 200
- repository status data renders normally
- infinite skeleton is gone
- error/retry state works
- GitHub module remains functional
- Super Admin gate remains functional
- production `/developer-hub` works
- normal OpenHands UI works
- restart survival passes

Changed file:

- `/opt/TBA/src/api/developer-hub-service.api.ts`
