---
theme: the-unnamed
title: Mock Auth Overview
info: |
  Documentation and explanation of the mock auth system
highlighter: shiki
drawings:
  persist: false
transition: slide-left
mdc: true
layout: cover
---

# Auth & Mocks

What's an auth?!🤔

---
transition: fade-out
layout: section
---

# Motivation

What lead to our current situation and need for deep infra changes

---
transition: slide-up
layout: default
---

# Motivations

<v-clicks>

- **Decoupling** The code shouldn't known about Auth0
- **Independence**  Avoid vendor lock-in
- **Flexibility**  Allow changing Auth provider "on the fly"
- **Design Ownership**  We should decide how our data is built, not a third party provider
- **Testing _(future)_**  We should be able to test our code without needing to mock the Auth0 SDK

</v-clicks>

---

# Implementation (Frontend)
Today, our code imports directly from the `auth0` library. This means that our code is tightly coupled to Auth0.

Making changes to the auth system is difficult and risky.
What if we want to change to a different provider? What if we want to change the way we handle tokens?

````md magic-move
```tsx {*|1}
import { useUser } from '@auth0/nextjs-auth0/client';

/// ... more code ...

function NavBar() {
  const { user } = useUser();

  return (
  <div>
    <Profile user={user} /> // This component takes a user and displays their name
  </div>
  )
}
```

```tsx {1|*}
import { useUser } from '@/common/hooks/use-user';

/// ... more code ...

function NavBar() {
  const { user } = useUser();

  return (
  <div>
    <Profile user={user} /> // This component takes a user and displays their name
  </div>
  )
}
```
````

---

# Implementation (Frontend)

Now that we saw how our components can stay unaware of the provenance of the user object, let's look at the implementation of `useUser` and its `Context` provider.

````md magic-move
```tsx
// The Provider is also unaware of the auth provider, but we had to create it
// to replace the Provider from `@auth0/nextjs-auth0`
import { userService } from '@/lib/services/user.service';

export const UserContext = createContext<UserContext>();

export const UserProvider = ({ children }: React.PropsWithChildren) => {
  const [state, setState] = useState<UserContext>({
    user: undefined,
    error: undefined,
    isLoading: true,
  });
  const { user, error } =  await loadUser();
  setState((previous) => ({ ...previous, user: user, error: error, isLoading: false }));

  return <UserContext.Provider value={value}>{children}</UserContext.Provider>;
};
```
```tsx {*|3}
export const userService = {
  getUser: async (): Promise<IUser> => {
    if (process.env.NEXT_PUBLIC_AUTH_MODE === 'mock') {
      return fetchMockUser();
    } else {
      return fetchAuth0User();
    }
  },
};
```
```tsx
const userFetcher = async (url: string) => {
  const response = await fetch(url);
  return response.json();
};

async function fetchAuth0User(): Promise<IUser> {
  const user = await userFetcher('/api/auth/me');
  return user as IUser;
}

function fetchMockUser(): IUser {
  return {
      name: 'Test User',
      email: 'test@test.com',
      org_id: 'test_org_id',
    };
}
```
````

---

# Implementation (Communication Frontend-Backend)

````md magic-move

```ts
// Example of an endpoint using the getApi function
export async function GET() {
  const api = await getApi();
  const allEngines = await api.engineControllerFindAllAdmin();

  return NextResponse.json({
    allEngines,
  });
}
```
```ts
import { getAccessToken } from '@/lib/api/get-access-token';

export async function getApi() {
  const accessToken = await getAccessToken();
  return new Api({
    baseUrl: process.env.NEXT_PUBLIC_MAIN_URL,
    baseApiParams: {
      headers: {
        Authorization: `Bearer ${accessToken.accessToken}`,
      },
    },
  }).v0;
}
```
```ts
import { getAccessToken as getAuth0AccessToken } from '@auth0/nextjs-auth0';

import { getMockAccessToken } from '@/lib/api/mock-access-token';

export async function getAccessToken() {
  if (process.env.NEXT_PUBLIC_AUTH_MODE === 'mock') {
    return getMockAccessToken();
  }
  return getAuth0AccessToken();
}
```
```ts
// Since our data is currently using Auth0 properties,
// we don't have a choice but to follow the same structure for now
const createMockJWT = () => {
  const payload = {
    name: 'Local User',
    email: 'local@test.com',
    org_id: 'org_local',
    ['https://www.apexsec.ai/user']: {
      email: 'local@test.com',
    },
    ['https://www.apexsec.ai/roles']: ['apex_admin', 'apex_user'],
  };

  return jwt.sign(payload, 'dev-secret', { expiresIn: '1h' });
};

export const getMockAccessToken = () => ({
  accessToken: createMockJWT(),
});

```
````
---

# Implementation (Backend)

The backend is also affected by the changes. We dynamically initialize a "simpler" JWT strategy when we use mocks.

````md magic-move
```ts
import { Module } from '@nestjs/common';

@Module({
  imports: [PassportModule.register({ defaultStrategy: 'jwt' })],
  providers: [JwtStrategy, TenantIdService],
  exports: [PassportModule, TenantIdService],
})
export class AuthzModule {}
```
```ts
import { DynamicModule, Module } from '@nestjs/common';

@Module({})
export class AuthzModule {
  static register(): DynamicModule {
    const providers = [TenantIdService, process.env.MOCK_AUTH === 'true' ? MockJwtStrategy : JwtStrategy];
    return {
      module: AuthzModule,
      imports: [PassportModule.register({ defaultStrategy: 'jwt' })],
      providers,
      exports: [PassportModule, TenantIdService],
    };
  }
}
```
```ts
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor() {
    super({
      secretOrKeyProvider: passportJwtSecret({
        cache: true,
        rateLimit: true,
        jwksRequestsPerMinute: 5,
        jwksUri: `${config.get('AUTH0_ISSUER_BASE_URL')}/.well-known/jwks.json`,
      }),

      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      audience: config.get('AUTH0_AUDIENCE'),
      issuer: `${config.get('AUTH0_ISSUER_BASE_URL')}/`, // Note the trailing slash!
      algorithms: ['RS256'],
    });
  }
}
```
```ts
export class MockJwtStrategy extends PassportStrategy(Strategy, 'jwt') {
  constructor() {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      secretOrKey: 'dev-secret',
      ignoreExpiration: true, // For mock purposes, ignore token expiration
    });
  }
}
```
````

---
transition: fade-out
layout: section
---

# Open Issues

Next steps and potential improvements

---
transition: slide-up
layout: default
---

# Open Issues

<v-clicks>

- **Duplication** Chat & Console have common components that are duplicated
- **Data Design** Our data has been designed by Auth0 format for now. How to improve this and make it provider agnostic?

</v-clicks>

---
layout: center
class: "text-center"
---

# Thank you✨
