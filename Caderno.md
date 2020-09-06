# AUTENTICAÇÃO / NODE - TYPESCRIPT (TYPEORM)

## 1 - CONFIGURAR TYPESCRIPT e NODEMON
- Iniciando a configuração do projeto
```
yarn init -y 
yarn add typescript ts-node nodemon -D 
```
- Para iniciar o typescript em nosso projeto 
```
npx typescript --init
```
- Adicionar script no package.json
```
"scripts": {
  "dev": "npx nodemon --exec ts-node src/index.ts"
},
```
- Criar pasta `src`
- Criar arquivo `index.ts` no path `/src`

## 2 - CONFIGURAR EXPRESS
- Instalar o express e os tipos typescript do express
```
yarn add express
yarn add @types/express -D
```

- Iniciar configuraçao do express no arquivo `index.ts`
```
import express from 'express';

const app = express();

app.use(express.json());

app.listen(3333, () => console.log(`Server started at http://localhost:3333`));
```

- Criar arquivo `routes.ts` no path `/src`
```
import { Router } from 'express';

const routes = Router();

export default routes;
```

- No arquivo `index.ts` adicionar/importar o arquivo de rotas
```
app.use(routes);
```

## 3 - CONFIGURAR TYPEORM
- Instalar o typeorm
```
yarn add typeorm reflect-metadata
```

- Importar o reflect-metadata em algum ponto global da aplicação, nesse caso no `./src/index.ts`. 
```
import 'reflect-metadata';
```

- Instalar o driver db 
```
yarn add pg
```

- Habilitar os decoretos em `tsconfig.json``
```
"strictPropertyInitialization": false,  /* Enable strict checking of property initialization in classes. */
.
.
.
"experimentalDecorators": true,        /* Enables experimental support for ES7 decorators. */
"emitDecoratorMetadata": true,         /* Enables experimental support for emitting type metadata for decorators. */
```

### INICIANDO CONFIGURAÇÃO DO TYPEORM
- Cria a pasta `models` em `./src/app/`
- Cria a pasta `migrations` em `./src/database/` 
- Criar um arquivo `ormconfig.json` na raiz do projeto
- Adicionar a configuração abaixo:
```
{
   "type": "postgres",
   "host": "localhost",
   "port": 5432,
   "username": "docker",
   "password": "docker",
   "database": "tsauth",
   "logging": true,
   "entities": [
      "src/app/models/*.ts"
   ],
   "migrations": [
      "src/database/migration/*.ts"
   ]
}
```

- Configurando CLI do typeorm no arquivo `package.json`
```
no scripts... {}, adicione:

"typeorm": "npx ts-node ./node_modules/typeorm/cli.js"
```

- Adicionar configuração do diretorio de criação das migrations no arquivo `ormconfig.json`
```
<codigo oculto>

"cli": {
  "migrationsDir": "src/database/migrations"
}
```

### ESTABELECENDO CONEXÃO COM O BANCO DE DADOS
- Criar arquivo `connect.ts` no path `/src/database`
```
import { createConnection } from 'typeorm';

createConnection().then(() => console.log('Successfully connected with database'));
```

- No arquivo `index.ts` adicionar/importar o arquivo de conexão
```
import './database/connect';
```

### AO FINAL DESSA FASE O `index.ts` DEVE TER ESSE CODE
```
import 'reflect-metadata';
import express from 'express';
import './database/connect';
import routes from './routes';

const app = express();

app.use(express.json());
app.use(routes);

app.listen(3333, () => console.log(`Server started at http://localhost:3333`));
```

## 4 - CRIANDO MIGRATIONS 
- Criando a migrations de usuario
```
yarn typeorm migration:create -n CreateUsersTable
```

- Editar a migration gerada 
```
   public async up(queryRunner: QueryRunner): Promise<void> {
        await queryRunner.query('CREATE EXTENSION IF NOT EXISTS "uuid-ossp"')

        await queryRunner.createTable(new Table({
            name: 'users',
            columns: [
                {
                    name: 'id',
                    type: 'uuid',
                    isPrimary: true,
                    generationStrategy: 'uuid',
                    default: 'uuid_generate_v4()',
                },
                {
                    name: 'email',
                    type: 'varchar',
                    isUnique: true
                },
                {
                    name: 'password',
                    type: 'varchar'
                },
            ]
        }))
    }

    public async down(queryRunner: QueryRunner): Promise<void> {
        await queryRunner.dropTable('users');
        await queryRunner.query('DROP EXTENSION "uuid-ossp"');
    }
```

- Execute a migration 
```
yarn typeorm migration:run
```

## 5 - CRIAR MODELS 
- Criando model de usuário
- Criar arquivo `User.ts` no path `/src/app/models`
```
import { Entity, PrimaryGeneratedColumn, Column } from 'typeorm';

@Entity('users')
class User {
  @PrimaryGeneratedColumn('increment')
  id: number;

  @Column()
  email: string;

  @Column()
  password: string;
}

export default User;
```

## 6 - CRIAR ROTAS 
- Criar arquivo `UserController.ts` no path `/src/app/controllers`
```
import { Request, Response } from "express";

class UserController {
  store(req: Request, res: Response) {
    return res.send('ok');
  }
}

export default new UserController();
```

- No arquivo `routes.ts` importe o controller `UserController.ts`
```
routes.post('/users', UserController.store);
```

- Criando um usuário de fato
    - Verificar se existe um usuário com o e-mail informado (code with changes)
```
import { Request, Response } from "express";
import { getRepository } from 'typeorm';
import User from "../models/User";

class UserController {
  async store(req: Request, res: Response) {
    const repository = getRepository(User);
    const { email, password } = req.body;

    const userExists = await repository.findOne(
      {
        where: {
          email
        }
      }
    );

    if (userExists) {
      return res.sendStatus(409);
    }

    const user = repository.create({ email, password });
    await repository.save(user);

    return res.json(user);
  }
}

export default new UserController();
```

- Instalar o bcryptjs 
```
yarn add bcryptjs
yarn add @types/bcryptjs -D
```

- Criar um metodo no modal
    - Metodo hashPassword no model `User.ts`
```
@BeforeInsert()
@BeforeUpdate()
    hashPassword() {
    this.password = bcrypt.hashSync(this.password, 8);
}
```

## 7 - ROTA DE AUTENTICAÇÃO
- Instalar o JWT 
```
yarn add jsonwebtoken
yarn add @types/jsonwebtoken -D
```

- Criar arquivo `AuthController.ts` no path `/src/app/controllers`
```
import { Request, Response } from "express";
import { getRepository } from 'typeorm';
import User from "../models/User";
import bcrypt from "bcryptjs";
import jwt from "jsonwebtoken";

class AuthController {
  async authenticate(req: Request, res: Response) {
    const repository = getRepository(User);
    const { email, password } = req.body;

    const user = await repository.findOne( { where: { email } } );

    if (!user) {
      return res.sendStatus(401);
    }

    const isValidPassword = await bcrypt.compare(password, user.password)
    
    if (!isValidPassword) {
      return res.sendStatus(401);
    }

    const token = jwt.sign({ id: user.id }, 'secret', { expiresIn: '1d' });
    
    return res.json({
      user, 
      token
    })
  }
}

export default new AuthController();
```

- Adicionar rota de auth no arquivo `routes.ts`
```
routes.post('/auth', AuthController.authenticate);
```

- Criar pasta `middleware` em `./src/app/`
- Criar arquivo `authMiddleware.ts` na pasta middlewares
```
import { Request, Response, NextFunction } from "express";
import jwt from "jsonwebtoken";

export default function authMiddleware(req: Request, res: Response, next: NextFunction) {
  const { authorization } = req.headers;

  if (!authorization) {
    return res.sendStatus(401);
  }

  const token = authorization.replace('Bearer', '').trim();

  try {
    const data = jwt.verify(token, 'secret');
    console.log(data);
  } catch {
    return res.sendStatus(401);
  }
}
```

- Colocar o middleware criado nas rotas que requerem autenticação no arquivo `routes.ts`
```
import { Router } from 'express';

import authMiddleware from './app/middlewares/authMiddleware';

import UserController from './app/controllers/UserController';
import AuthController from './app/controllers/AuthController';

const routes = Router();

routes.post('/users', UserController.store);
routes.post('/auth', AuthController.authenticate);
routes.get('/users', authMiddleware, UserController.index);

export default routes;
```