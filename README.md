## NextJS-api
### [schema.prisma](https://github.com/Omarmdwasimuddin/Prisma-Schema-Many-to-Many-Relation)
### lib/prisma.ts
```
import { PrismaClient } from "@/generated/prisma";
import { PrismaPg } from "@prisma/adapter-pg";
import pg from "pg";

const connectionString = process.env.DATABASE_URL!;
const pool = new pg.Pool({ connectionString });
const adapter = new PrismaPg(pool);

const globalForPrisma = globalThis as unknown as { prisma: PrismaClient };

export const prisma =
  globalForPrisma.prisma ?? new PrismaClient({ adapter });

if (process.env.NODE_ENV !== "production") globalForPrisma.prisma = prisma;

```

### Install koro
```bash
npm install zod bcryptjs
npm install -D @types/bcryptjs
```
---

### api/user/route.ts

```
import { prisma } from "@/lib/prisma";
import { NextResponse } from "next/server";
import { z } from "zod";  // Schema validation er jonno
import bcrypt from "bcryptjs"; // Password hashing er jonno

// Validation
const UserCreateSchema = z.object({
    email: z.string().trim().toLowerCase().email("Invalid email address"),
    password: z.string().min(8, "Password must be at least 8 characters long"),
});

export async function POST(request: Request) {
    try {

        const json = await request.json();
        const body = UserCreateSchema.parse(json); // Validate incoming data

        // Check if user already exists
        const existingUser = await prisma.user.findUnique({
            where: { email: body.email },
        });

        if (existingUser) {
            return NextResponse.json(
                { message: "User already exists" },
                { status: 409 } //Conflict
            );
        }

        // Hash the password before saving
        const hashedPassword = await bcrypt.hash(body.password, 10);

        // Create new user
        const newUser = await prisma.user.create({
            data: {
                email: body.email,
                password: hashedPassword,
            },
            // Select only necessary fields to return for security reasons
            select: {
                id: true,
                email: true,
                createdAt: true,
            },
        });

        return NextResponse.json(
            { status: "success", data: newUser },
            { status: 201 } // Created
        );

    } catch (error) {
        // Zod validation error handle kora
        if (error instanceof z.ZodError) {
            return NextResponse.json(
                { status: "error", message: "Validation failed", errors: error.issues },
                { status: 400 }
            );
        }

        console.error("User create error:", error);
        return NextResponse.json(
            { status: "error", message: "Internal server error" },
            { status: 500 }
        );
    }
}
```
---
