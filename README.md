# jest-prisma

[![github actions](https://github.com/Quramy/jest-prisma/workflows/build/badge.svg)](https://github.com/Quramy/jest-prisma/actions)
[![npm version](https://badge.fury.io/js/@quramy%2Fjest-prisma.svg)](https://badge.fury.io/js/@quramy%2Fjest-prisma)
[![GitHub license](https://img.shields.io/badge/license-MIT-blue.svg)](https://raw.githubusercontent.com/Quramy/jest-prisma/main/LICENSE.txt)

Jest environment for Prisma integrated testing.
You can run each test case in isolated transaction which is rolled back automatically.

## How to use

#### Install

```sh
$ npm i @quramy/jest-prisma -D
```

#### Configure Jest

```js
/* jest.config.mjs */
export default {
  // ... Your jest configuration

  testEnvironment: "@quramy/jest-prisma/environment",
};
```

#### Configure TypeScript

```js
/* tsconfig.json */

{
  "compilerOptions": {
    "types": ["@types/jest", "@quramy/jest-prisma"],
  }
}
```

#### Configure Prisma

jest-prisma uses [Prisma interactive transaction feature](https://www.prisma.io/docs/concepts/components/prisma-client/transactions#interactive-transactions-in-preview). Edit your schema.prisma and turn `interactiveTransactions` on.

```gql
generator client {
  provider        = "prisma-client-js"
  previewFeatures = ["interactiveTransactions"]
}
```

#### Write tests

Global object `jestPrisma` is provided within jest-prisma environment. And Prisma client instance is available via `jestPrisma.client`

```ts
describe(UserService, () => {
  // jestPrisma.client works with transaction rolled-back automatically after each test case end.
  const prisma = jestPrisma.client;

  test("Add user", async () => {
    const createdUser = await prisma.user.create({
      data: {
        id: "001",
        name: "quramy",
      },
    });

    expect(
      await prisma.user.findFirst({
        where: {
          name: "quramy",
        },
      }),
    ).toStrictEqual(createdUser);
  });

  test("Count user", async () => {
    expect(await prisma.user.count()).toBe(0);
  });
});
```

## Configuration

You can pass some options using `testEnvironmentOptions`.

```js
/* jest.config.mjs */
export default {
  testEnvironment: "@quramy/jest-prisma/environment",
  testEnvironmentOptions: {
    verboseQuery: true,
  },
};
```

Alternatively, you can use `@jest-environment-options` pragma in your test file:

```js
/**
 *
 * @jest-environment-options: { "verboseQuery": true }
 *
 */
test("it should execute prisma client", () => {
  /* .... */
});
```

## Tips

### Singleton

If your project uses singleton Prisma client instance, such as:

```ts
/* src/client.ts */
import { PrismaClient } from "@prisma/client";

export const prisma = new PrismaClient();
```

```ts
/* src/userService.ts */

import { prisma } from "./client.ts";

export function findUserById(id: string) {
  const result = await prisma.user.findUnique({
    where: { id },
  });
  return result;
}
```

You can replace the singleton instance to `jestPrisma.client` via `jest.mock`.

```js
/* setup-prisma.js */

jest.mock("./src/client", () => {
  return {
    prisma: jestPrisma.client,
  };
});
```

```js
/* jest.config.mjs */
export default {
  testEnvironment: "@quramy/jest-prisma/environment",
  setupFilesAfterEnv: ["<rootDir>/setup-prisma.js"],
};
```

```ts
import { prisma } from "./client";

import { findUserById } from "./userService";

describe("findUserById", () => {
  beforeEach(async () => {
    await prisma.user.create({
      data: {
        id: "test_user_id",
      },
    });
  });

  it("should return user", async () => {
    await findUserById("test_user_id");
    // assertion
  });
});
```

### DI Containers

If you're using DI containers such as [InversifyJS](https://github.com/inversify/InversifyJS) or [Awilix](https://github.com/jeffijoe/awilix) and wish to introduce jest-prisma, you can easily do that just by rebinding PrismaClient to a global `jestPrisma` instance provided by jest-prisma.

Here is an example below. Given that we have the following repository. Note that it is decorated by `@injectable` so will `prisma` will be inject as a constructor argument.

```ts
/* types.ts */
export const TYPES = {
  PrismaClient: Symbol.for("PrismaClient"),
  UserRepository: Symbol.for("UserRepository"),
};
```

```ts
/* user-repository.ts */
import { TYPES } from "./types";

interface IUserRepository {
  findAll(): Promise<User[]>;
  findById(): Promise<User[]>;
  save(): Promise<User[]>;
}

@injectable()
class UserRepositoryPrisma implements IUserRepository {
  constructor(
    @inject(TYPES.PrismaClient)
    private readonly prisma: PrismaClient,
  ) {}

  async findAll() { .. }

  async findById() { .. }

  async save() { .. }
}
```

```ts
/* inversify.config.ts */
import { Container } from "inversify";
import { PrismaClient } from "prisma";

import { TYPES } from "./types";
import { UserRepositoryPrisma, IUserRepository } from "./user-repository";

const container = new Container();

container.bind(TYPES.PrismaClient).toConstantValue(new PrismaClient());

container.bind<IUserRepository>(TYPES.UserRepository).to(UserRepositoryPrisma);
```

In most cases, the setup above allows you to inject a pre-configured `PrismaClient` by associating the symbol to an actual instance like `bind(TYPES.PrismaClient).toConstantValue(new PrismaClient())` and then acquire the repository by `get(TYPES.UserRepository)`.

However, with jest-prisma, the global `jestPrisma.client` object is initialised for each unit tests so you have to make sure that you're binding the instance _after_ the initialisation.

Note that we're rebinding PrismaClient to the jest-prisma inside `beforeEach` phase. Any other phase including `beforeAll` or `setupFilesAfterEnv` may not work as you expect.

```ts
/* user-repository.spec.ts */
describe("UserRepository", () => {
  beforeEach(() => {
    container
      .rebind(TYPES.PrismaClient)
      .toConstantValue(jestPrisma.client);
  });

  it("creates a user" ,() => {
    constainer.get<IUserRepository>(TYPES.UserRepository);
    ...
  });
});
```

## References

### `global.jestPrisma`

```ts
export interface JestPrisma {
  /**
   *
   * Primsa Client Instance whose transaction are isolated for each test case.
   * And this transaction is rolled back automatically after each test case.
   *
   */
  readonly client: PrismaClient;
}
```

### Environment options

```ts
export interface JestPrismaEnvironmentOptions {
  /**
   *
   * If set true, each transaction is not rolled back but committed.
   *
   */
  readonly disableRollback?: boolean;

  /**
   *
   * Display SQL queries in test cases to STDOUT.
   *
   */
  readonly verboseQuery?: boolean;
}
```

## License

MIT
