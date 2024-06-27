# Sequenciamento Node JS com typescript - Node / Fastify / knex

1: npm init -y (vai criar o arquivo package.json)
2: npm i -D typescript (instalar o typescript) / Após criar o arquivo de config do Typescript: npx tsc --init (Será criado o arquivo tsconfig.json)
3: mudar o target do tsconfig para "es2020"
4: executar a compatibilidade do node com o typescript: > npm install -D @types/node
5: criar o arquivo Server.ts e instalar o fastify: npm i fastify
6: criar a primeira rota no arquivo Server.ts
    # app.get => GET busca o valor
    # app.listen => adiciona uma porta e verifica se o servidor está funcionando.
7: automatizar o servidor npm install tsx -D. criar o script watch no arquivo package.json -> "dev": "tsx src/server.ts"
8: criar o arquivo database.ts na pasta SRC e rodar o npm install knex sqlite3 ou pg para o banco de dados.
9: no arquivo database.ts devo importar o knex ( import {knex as setupKnex} from "knex";) //// adicionar " , " e Knex eu coloco a interface no knex e posso colocar as regras do mesmo, por exemplo mudar o diretório: 

        extension: 'ts',
        directory: './db/migrations',

10: no arquivo database.ts devo exportar um config que é o acesso ao banco: 
#    [export const config: Knex.Config = ({
#           client: 'sqlite',
#        connection: {
#        filename: './tmp/app.db'
#    },

#    useNullAsDefault: true,
#  migrations: {
  #      extension: 'ts',
   #     directory: './db/migrations',
   # } })

# export const knex = setupKnex(config)]

11: no knexfile.ts: importar o config: 

# import { config } from './src/database' export default config

12: criar um script no package json: "knex": "node --import tsx ./node_modules/.bin/knex"
     
13: executar: npm run knex migrate:make "nome_banco" parar criar o migration

14: começar as tabelas do banco, criar nas migrations

15: metodo UP para criar a tabela e down excluir.

16: para criar as tabelas rodar no terminal: npm run knex -- migrate:latest // para desfazer a migration: npm run knex -- migrate:rollback

17: criar variaveis de ambiente: instalar a extensão dotenv. Criar um arquivo .env e lá colocar os caminhos: exemplo: DATABASE_CLIENT = sqlite
                                                                                                                      DATABASE_URL="./db/app.db"
                                                                                                                      NODE_ENV=development

18: importar para o arquivo database: import 'dotenv/config' e depois: colocar no código  filename: process.env.DATABASE_URL,

19: USAR O ZOD PARA VALIDAÇÃO DE DADOS. Criar uma pasta .env na .src e depois um arquivo index.ts

20: instalar a biblioteca zod. -> npm i zod

21: tirar o import dotenv/config do database e passar para o index. importar também o zod import { z } from 'zod'

22 no arquivo index.ts fazer a validação: preciso exportar essa validação e exportar onde usar para isso, ta na ultima linha ali do codigo

# const envSchema = z.object({
  #  NOVE_ENV: z.enum(['development', 'test', 'production']).default('production'),
   # DATABASE_URL: z.string(),
   # PORT: z.number().default(3333),
# })

# const _env = envSchema.safeParse(process.env)

# if (_env.success === false ) {
  #  console.error('Invalid environment variables', _env.error.format())

   # throw new Error ('Invalid environmente variables.')
# })

- depois adicionar no .env e no .env.example

23 importar a variavel env onde usar, no database por exemplo: import { env } from "./env";

24: criar plugins e separar a app em partes. clicar na src a pasta routes e separar elas:

25: no arquivo routes criar as rotas pro banco de dados insere os valores etc /// '/:mealId' parametro unico

import { FastifyInstance } from 'fastify'
import { z } from 'zod'
import { checkSessionIdExists } from '../middlewares/check-session-id-exists'
import { randomUUID } from 'node:crypto'
import { knex } from '../database'

export async function mealsRoutes(app: FastifyInstance) {
  app.post(
    '/',
    { preHandler: [checkSessionIdExists] },
    async (request, reply) => {
      const createMealBodySchema = z.object({
        name: z.string(),
        description: z.string(),
        isOnDiet: z.boolean(),
        date: z.coerce.date(),
      })

      const { name, description, isOnDiet, date } = createMealBodySchema.parse(
        request.body,
      )

      await knex('meals').insert({
        id: randomUUID(),
        name,
        description,
        is_on_diet: isOnDiet,
        date: date.getTime(),
        user_id: request.user?.id,
      })

      return reply.status(201).send()
    },
  )

  app.get(
    '/',
    { preHandler: [checkSessionIdExists] },
    async (request, reply) => {
      const meals = await knex('meals')
        .where({ user_id: request.user?.id })
        .orderBy('date', 'desc')

      return reply.send({ meals })
    },
  )

  app.get(
    '/:mealId',
    { preHandler: [checkSessionIdExists] },
    async (request, reply) => {
      const paramsSchema = z.object({ mealId: z.string().uuid() })

      const { mealId } = paramsSchema.parse(request.params)

      const meal = await knex('meals').where({ id: mealId }).first()

      if (!meal) {
        return reply.status(404).send({ error: 'Meal not found' })
      }

      return reply.send({ meal })
    },
  )

  app.put(
    '/:mealId',
    { preHandler: [checkSessionIdExists] },
    async (request, reply) => {
      const paramsSchema = z.object({ mealId: z.string().uuid() })

      const { mealId } = paramsSchema.parse(request.params)

      const updateMealBodySchema = z.object({
        name: z.string(),
        description: z.string(),
        isOnDiet: z.boolean(),
        date: z.coerce.date(),
      })

      const { name, description, isOnDiet, date } = updateMealBodySchema.parse(
        request.body,
      )

      const meal = await knex('meals').where({ id: mealId }).first()

      if (!meal) {
        return reply.status(404).send({ error: 'Meal not found' })
      }

      await knex('meals').where({ id: mealId }).update({
        name,
        description,
        is_on_diet: isOnDiet,
        date: date.getTime(),
      })

      return reply.status(204).send()
    },
  )

  app.delete(
    '/:mealId',
    { preHandler: [checkSessionIdExists] },
    async (request, reply) => {
      const paramsSchema = z.object({ mealId: z.string().uuid() })

      const { mealId } = paramsSchema.parse(request.params)

      const meal = await knex('meals').where({ id: mealId }).first()

      if (!meal) {
        return reply.status(404).send({ error: 'Meal not found' })
      }

      await knex('meals').where({ id: mealId }).delete()

      return reply.status(204).send()
    },
  )

  app.get(
    '/metrics',
    { preHandler: [checkSessionIdExists] },
    async (request, reply) => {
      const totalMealsOnDiet = await knex('meals')
        .where({ user_id: request.user?.id, is_on_diet: true })
        .count('id', { as: 'total' })
        .first()

      const totalMealsOffDiet = await knex('meals')
        .where({ user_id: request.user?.id, is_on_diet: false })
        .count('id', { as: 'total' })
        .first()

      const totalMeals = await knex('meals')
        .where({ user_id: request.user?.id })
        .orderBy('date', 'desc')

      const { bestOnDietSequence } = totalMeals.reduce(
        (acc, meal) => {
          if (meal.is_on_diet) {
            acc.currentSequence += 1
          } else {
            acc.currentSequence = 0
          }

          if (acc.currentSequence > acc.bestOnDietSequence) {
            acc.bestOnDietSequence = acc.currentSequence
          }

          return acc
        },
        { bestOnDietSequence: 0, currentSequence: 0 },
      )

      return reply.send({
        totalMeals: totalMeals.length,
        totalMealsOnDiet: totalMealsOnDiet?.total,
        totalMealsOffDiet: totalMealsOffDiet?.total,
        bestOnDietSequence,
      })
    },
  )
}

26. criar a pasta @types, na pasta src. e tipar os codigos do knex/fastify. criar o arquivo: knex.d.ts

exemplo do código nesse arquivo: import 'knex'

declare module 'knex/types/tables' {
  export interface Tables {


    27 -> adicionar cookies, manter contexto entre requisições. instalar no terminal:  npm i @fastify/cookie

    28 -> criar pasta middlewares para validar o id

    29 -> iniciar testes, ou fazendo peuqenos testes durante a aplicação. utilzar o vitest