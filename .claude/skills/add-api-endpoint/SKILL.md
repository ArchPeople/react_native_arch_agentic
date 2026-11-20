---
name: add-api-endpoint
description: Adds a new REST API call following the repo's layered networking pattern - endpoint path in api-path.ts, base URL in config, request/response interfaces, a data-source object in src/core/data/remote that wraps apiFetch (axios) and returns IBaseFailure instead of throwing, and a react-query hook (useQuery/useMutation) in the feature's api folder. Also covers auth headers via axios interceptors and cache invalidation. Use when integrating a backend endpoint, fetching or posting data, adding a GET/POST/PUT/DELETE, or changing headers/interceptors. Triggers: API, endpoint, fetch, axios, request, response, react-query, useQuery, useMutation, interceptor, token.
---

# Add an API endpoint

Canonical example:
- `src/core/data/remote/demo-data/demoData.ts`
- `src/features/demo-feature/api/demoFeatureApi.ts`

Flow: **screen → feature api hook → core data source → `apiFetch` → axios**.

## 1. Path and base URL
`src/core/data/remote/api-path.ts`:
```ts
export const apiPath = {
  demoEndpoint: '/movies.json',
  orderDetailEndpoint: '/orders',
};
```
The base URL lives in `src/core/config/config.ts`. If it differs per flavor, source it from `env` (see `manage-environment-config`):
```ts
export const config = {
  demoDomain: 'https://reactnative.dev',
  apiDomain: env.api_domain,
  flavor: env.flavor,
};
```

## 2. Interfaces
`src/core/data/remote/<name>-data/interface/<name>Response.interface.ts` (and `<name>Request.interface.ts` if the call has a body or params):
```ts
export interface IOrderDetailResponse {
  id: string;
  items: IOrderDetailResponseItem[];
}

export interface IOrderDetailResponseItem {
  sku: string;
  qty: number;
}
```

## 3. Data source (never throws)
`src/core/data/remote/<name>-data/<name>Data.ts`: an exported object literal of async methods returning `Promise<IResponse | IBaseFailure>`:
```ts
import { config } from '@core/config';
import { apiFetch } from '@core/services/api-fetch';
import { apiPath } from '../api-path';
import { IBaseFailure } from '@core/common/base/baseFailure.interface';
import { GenericAbortSignal } from 'axios';
import { IOrderDetailResponse } from './interface/orderDetailResponse.interface';

interface IFetchOrderDetailParams {
  id: string;
  signal?: GenericAbortSignal;
}

export const orderDetailData = {
  fetchOrderDetail: async ({
    id,
    signal,
  }: IFetchOrderDetailParams): Promise<IOrderDetailResponse | IBaseFailure> => {
    try {
      const data: IOrderDetailResponse = await apiFetch.getApi(
        config.apiDomain + apiPath.orderDetailEndpoint + '/' + id,
        { signal },
      );
      return data;
    } catch (error: any) {
      const failure: IBaseFailure = {
        errorStatus: error.response?.status ?? 0,
        errorCode: error.code,
        errorMessage: error.message,
      };
      return failure;
    }
  },
};
```
- Use `apiFetch.postApi(url, { body, params, headers, signal })`, and `putApi`/`deleteApi` the same way (`src/core/services/api-fetch/interface/apiFetch.interface.ts`).
- Use `error.response?.status`. The demo uses `error.response.status`, which crashes on network errors and timeouts because there is no response object.
- Export the object from `src/core/data/remote/index.ts`:
  ```ts
  export { orderDetailData } from './order-detail-data/orderDetailData';
  ```

## 4. Feature hook (react-query)
`src/features/<feature>/api/<camel>Api.ts`:
```ts
import { useMutation, useQuery, useQueryClient } from '@tanstack/react-query';
import { orderDetailData } from '@core/data/remote';
import { IOrderDetailResponse } from '@core/data/remote/order-detail-data/interface/orderDetailResponse.interface';

export const useGetOrderDetail = (id: string) => {
  return useQuery({
    queryKey: ['orderDetail', id],
    queryFn: async ({ signal }): Promise<IOrderDetailResponse> => {
      const response = await orderDetailData.fetchOrderDetail({ id, signal });
      if ('errorStatus' in response) {
        throw new Error(response.errorMessage);
      }
      return response;
    },
  });
};

export const useCancelOrder = () => {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: async (id: string) => {
      const response = await orderDetailData.cancelOrder({ id });
      if ('errorStatus' in response) {
        throw new Error(response.errorMessage);
      }
      return response;
    },
    onSuccess: (_, id) => {
      queryClient.invalidateQueries({ queryKey: ['orderDetail', id] });
    },
  });
};
```
- Query keys must be unique per resource. Plop's template copies `['demoData']`, so always change it.
- Always pass `signal` through, so react-query can cancel requests.
- The global client is `src/core/services/query/query.ts` (`retry: 0`). Override per query (`retry`, `staleTime`, `enabled: !!id`) rather than changing the defaults.

## 5. Use it in the screen
```tsx
const { data, isFetching, isError, error, refetch } = useGetOrderDetail(id);
```
Show loading, error and empty states. Screens never import `apiFetch` or `axios`.

## Auth headers, logging and refresh
Add them **once** in the interceptors in `src/core/services/api-fetch/apiFetch.ts`:
```ts
api.interceptors.request.use((config: InternalAxiosRequestConfig) => {
  const token = sessionData.getToken();
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});
```
Read the token from a `core/data/local` module (see `add-local-storage`). Never hard-code tokens or API keys.

## Verify
```bash
npx prettier --check src/core/data/remote src/features/<feature>/api
npx tsc --noEmit --ignoreDeprecations 6.0
```
Then add a data-source test that mocks `@core/services/api-fetch` and checks the success and `IBaseFailure` mapping (see `write-tests`), and run it.
