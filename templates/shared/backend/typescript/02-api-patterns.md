# TypeScript API実装パターン

## Express.js
```typescript
import express, { Request, Response, NextFunction } from 'express';
import { body, validationResult } from 'express-validator';

interface User {
  id: number;
  name: string;
  email: string;
}

const app = express();

// ミドルウェア
app.use(express.json());
app.use(express.urlencoded({ extended: true }));

// ルーティング
app.get('/api/users', getUsers);
app.post('/api/users', 
  body('name').isLength({ min: 2 }),
  body('email').isEmail(),
  createUser
);

async function getUsers(req: Request, res: Response) {
  try {
    const users: User[] = await userService.getAll();
    res.json({
      success: true,
      data: users
    });
  } catch (error) {
    res.status(500).json({
      success: false,
      error: 'Internal server error'
    });
  }
}

async function createUser(req: Request, res: Response) {
  const errors = validationResult(req);
  if (!errors.isEmpty()) {
    return res.status(400).json({
      success: false,
      errors: errors.array()
    });
  }

  const user = await userService.create(req.body);
  res.status(201).json({
    success: true,
    data: user
  });
}
```

## Fastify
```typescript
import fastify from 'fastify';

const server = fastify({ logger: true });

// スキーマ定義
const userSchema = {
  type: 'object',
  required: ['name', 'email'],
  properties: {
    name: { type: 'string', minLength: 2 },
    email: { type: 'string', format: 'email' }
  }
};

// ルート登録
server.post<{ Body: CreateUserDto }>('/api/users', {
  schema: { body: userSchema }
}, async (request, reply) => {
  const user = await userService.create(request.body);
  return { success: true, data: user };
});

server.listen({ port: 3000 });
```