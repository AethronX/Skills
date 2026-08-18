# Additional CLI Tools

Less common but useful CLIs for specific workflows.

## AWS CLI (`aws`)

Core commands for web developers (S3, CloudFront, Lambda):

```bash
# S3
aws s3 cp ./dist s3://bucket-name --recursive
aws s3 sync ./dist s3://bucket-name --delete
aws s3 ls s3://bucket-name

# CloudFront
aws cloudfront create-invalidation --distribution-id ID --paths "/*"

# Lambda
aws lambda invoke --function-name my-func output.json
aws lambda update-function-code --function-name my-func --zip-file fileb://func.zip

# Secrets Manager
aws secretsmanager get-secret-value --secret-id my-secret

# SSM Parameter Store
aws ssm get-parameter --name /app/db-url --with-decryption
```

## Railway (`railway`)

```bash
railway login
railway init
railway up              # deploy
railway run <cmd>       # run command with env vars
railway logs
railway variables       # list env vars
railway variables set KEY=value
railway connect postgres  # connect to db
```

## Fly.io (`flyctl`)

```bash
fly launch              # create app
fly deploy
fly status
fly logs
fly secrets set KEY=value
fly ssh console         # SSH into machine
fly postgres create
fly proxy 5432 -a my-db # tunnel to postgres
```

## ngrok

Expose localhost for webhook testing:

```bash
ngrok http 3000
ngrok http 3000 --domain=my-app.ngrok.io  # custom domain (paid)
```

Useful for:
- Stripe webhook testing
- OAuth callback testing
- Mobile device testing

## Turborepo (`turbo`)

Monorepo build orchestration:

```bash
turbo run build
turbo run build --filter=@app/web
turbo run dev --parallel
turbo run lint test build  # run in dependency order
turbo prune --scope=@app/web  # create minimal monorepo
```

## Planetscale (`pscale`)

Serverless MySQL:

```bash
pscale auth login
pscale connect <db> <branch> --port 3309
pscale branch create <db> <branch>
pscale deploy-request create <db> <branch>
pscale deploy-request deploy <db> <number>
```

## Neon (`neonctl`)

Serverless Postgres:

```bash
neonctl auth
neonctl projects list
neonctl branches create --name dev
neonctl connection-string
neonctl databases create --name mydb
```

## Drizzle (`drizzle-kit`)

TypeScript ORM (alternative to Prisma):

```bash
npx drizzle-kit generate
npx drizzle-kit push
npx drizzle-kit migrate
npx drizzle-kit studio
```

## Bun

All-in-one JS runtime and package manager:

```bash
bun init
bun install
bun add <package>
bun run <script>
bun test
bun build ./index.ts --outdir ./dist
bunx <package>          # like npx
```
