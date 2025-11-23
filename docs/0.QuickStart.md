# QuickStart

## Initial Source `ecommerce-frontend`

### Tech Stack
- Framework: NextJS
- Fetch data: Axios, Tanstack Query
- Style: TailwindCSS + Shadcn/ui

### Install and Run
- Initial source: `npx create-next-app@latest ecommerce-frontend`
- Run: 
```
cd ecommerce-frontend
npm run dev
```

## Initial Source `ecommerce-backend`

### Tech Stack
- Framework: NestJS
- Database: Supabase - PostgreSQL
- ORM: TypeORM
- Auth: Better Auth
- File manage: Supabase Storage

### Install and Run
- Initial source: 
```
npm i -g @nestjs/cli
nest new ecommerce-backend
```
- Run: 
```
cd ecommerce-backend
npm run start:dev
```