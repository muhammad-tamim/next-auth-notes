<h1 align="center">Auth.js Notes</h1>

- [Installation:](#installation)
- [Initial Setup:](#initial-setup)
- [Implement social login and credential login:](#implement-social-login-and-credential-login)
- [Protect routes by middleware:](#protect-routes-by-middleware)
- [protect api:](#protect-api)
- [Example:](#example)

Auth.js is a modern open-source authentication library for JavaScript applications, especially popular in the Next.js ecosystem. It was previously known as NextAuth.js. The rename happened because the project expanded beyond just Next.js and now supports multiple frameworks like express, SvelteKit, and Qwik.

# Installation:

```bash
npm install next-auth@beta
npm install bcryptjs
```

```bash
// .env
MONGO_URI=mongodb://localhost:27017/
MONGO_DB_NAME=testDB
NEXTAUTH_SECRET=supersecretkey
GOOGLE_CLIENT_ID=................
GOOGLE_CLIENT_SECRET=..........
GITHUB_ID=...........
GITHUB_SECRET=............
```

# Initial Setup: 

```ts
// src/auth.ts

import NextAuth from "next-auth";
import GoogleProvider from "next-auth/providers/google";
import GitHubProvider from "next-auth/providers/github";
import CredentialsProvider from "next-auth/providers/credentials";

import bcrypt from "bcryptjs";
import { dbConnect } from "@/lib/dbConnect";

export const { handlers, signIn, signOut, auth } = NextAuth({
    providers: [
        GoogleProvider({
            clientId: process.env.GOOGLE_CLIENT_ID!,
            clientSecret: process.env.GOOGLE_CLIENT_SECRET!,

            authorization: {
                params: {
                    prompt: "consent",
                    access_type: "offline",
                    response_type: "code",
                },
            },
        }),

        GitHubProvider({
            clientId: process.env.GITHUB_ID!,
            clientSecret: process.env.GITHUB_SECRET!,
        }),

        CredentialsProvider({
            name: "credentials",

            credentials: {
                email: {},
                password: {},
            },

            async authorize(credentials) {
                const email = credentials.email as string;
                const password = credentials.password as string;

                if (!email || !password) {
                    throw new Error("Missing credentials");
                }

                const db = await dbConnect();
                const usersCollection = db.collection("users");

                const user = await usersCollection.findOne({ email });

                if (!user) {
                    throw new Error("User not found");
                }

                const passwordMatch = await bcrypt.compare(password, user.password);

                if (!passwordMatch) {
                    throw new Error("Invalid password");
                }

                return {
                    id: user._id.toString(),
                    name: user.name,
                    email: user.email,
                    image: user.image,
                    role: user.role,
                };
            },
        }),
    ],

    session: {
        strategy: "jwt",
    },

    pages: {
        signIn: "/sign-in",
    },

    callbacks: {

        async signIn({ user, account }) {

            // only for google/github login
            if (account?.provider === "google" || account?.provider === "github") {

                const db = await dbConnect();
                const usersCollection = db.collection("users");

                const existingUser = await usersCollection.findOne({ email: user.email });

                // create user if not exists
                const newUser = {
                    name: user.name,
                    email: user.email,
                    image: user.image,
                    role: "user",
                    provider: account.provider,
                    createdAt: new Date(),
                };
                if (!existingUser) {
                    await usersCollection.insertOne(newUser);
                }
            }

            return true;
        },

        async jwt({ token, user }) {

            const db = await dbConnect();
            const usersCollection = db.collection("users");

            // first login
            if (user?.email) {

                const dbUser = await usersCollection.findOne({ email: user.email });

                if (dbUser) {
                    token.role = dbUser.role;
                }
            }

            return token;
        },

        async session({ session, token }) {

            if (session.user) {
                session.user.role = token.role as string;
            }

            return session;
        },
    },
});
```

```ts
// src/app/api/auth/[...nextauth]/route.ts

import { handlers } from "@/auth";

export const { GET, POST } = handlers
```

```ts
// src/lib/dbConnect.ts

import { MongoClient, ServerApiVersion } from "mongodb"

const uri = process.env.MONGO_URI as string
const dbName = process.env.MONGO_DB_NAME as string

const client = new MongoClient(uri, {
    serverApi: {
        version: ServerApiVersion.v1,
        strict: true,
        deprecationErrors: true,
    },
})

export async function dbConnect() {

    await client.connect()
    return client.db(dbName)
}
```

```ts
// src/types/next-auth.d.ts

import "next-auth";
import "next-auth/jwt";

declare module "next-auth" {
    interface Session {
        user: {
            role?: string;
        } & DefaultSession["user"];
    }
}

declare module "next-auth/jwt" {
    interface JWT {
        role?: string;
    }
}
```

# Implement social login and credential login: 


```ts
// src/app/layout.tsx
import type { Metadata } from "next";
import "./globals.css";
import Navbar from "@/components/Navbar";


export const metadata: Metadata = {
  title: "Create Next App",
  description: "Generated by create next app",
};

export default function RootLayout({
  children,
}: Readonly<{
  children: React.ReactNode;
}>) {
  return (
    <html lang="en">
      <body className="min-h-full flex flex-col">
        <Navbar></Navbar>
        {children}
      </body>
    </html>
  );
}
```

```ts
// src/app/page.tsx

import { auth } from '@/auth'
import React from 'react'

export default async function HomePage() {
  const session = await auth()
  const { name, email } = session?.user || {}
  return (
    <div className='text-center'>
      <h1 className='text-5xl font-semibold'>Home Page</h1>
      <div className='border p-2 m-5 max-w-md mx-auto'>
        <h1 className='text-2xl'>User Info:</h1>
        <p>name: {name}</p>
        <p>email: {email}</p>
      </div>

    </div>
  )
}
```

```ts
// src/actions/auth.action.ts
"use server";

import bcrypt from "bcryptjs";
import { dbConnect } from "@/lib/dbConnect";
import { signIn, signOut } from "@/auth";

export async function registerUser(formData: FormData) {
    const name = formData.get("name") as string;
    const image = formData.get("image") as string;
    const email = formData.get("email") as string;
    const password = formData.get("password") as string;

    const db = await dbConnect();
    const usersCollection = db.collection("users");

    const existingUser = await usersCollection.findOne({ email });

    if (existingUser) {
        throw new Error("User already exists");
    }

    const hashedPassword = await bcrypt.hash(password, 10);

    const user = {
        name,
        image,
        email,
        password: hashedPassword,
        provider: "credentials",
        role: "user",
        createdAt: new Date(),
    };

    await usersCollection.insertOne(user);

    await signIn("credentials", {
        email,
        password,
        redirectTo: "/",
    });
}

export async function loginUser(formData: FormData) {
    const email = formData.get("email") as string;
    const password = formData.get("password") as string;

    await signIn("credentials", {
        email,
        password,
        redirectTo: "/",
    });
}

export async function doSocialLogin(formData: FormData) {
    const action = formData.get("action") as string;

    await signIn(action, {
        redirectTo: "/",
    });
}

export async function doLogout() {
    await signOut({
        redirectTo: "/",
    });
}
```

```ts
// src/app/sign-in/page.tsx

import SignInForm from '@/components/SignInForm'
import React from 'react'

export default function SignInPage() {
    return (
        <div>
            <SignInForm></SignInForm>
        </div>
    )
}
```

```ts
// src/app/sign-up/page.tsx

import SignUpForm from '@/components/SignUpForm'
import React from 'react'

export default function SignUpPage() {
    return (
        <div>
            <SignUpForm></SignUpForm>
        </div>
    )
}
```

```ts
// src/components/Navbar.tsx

import Image from 'next/image'
import Link from 'next/link'
import React from 'react'
import SignOutButton from './SignOutButton'
import { auth } from '@/auth'

export default async function Navbar() {
    const session = await auth()
    return (
        <div className='flex justify-end items-center gap-3 p-2 border mb-5'>
            <Link href={'/'}>Home</Link>
            <Link href={'/products'}>Products</Link>
            <Link href={'/secret'}>Secret</Link>

            {
                !session?.user
                    ?
                    <>
                        <Link href={'/sign-in'}>Sign In</Link>
                        <Link href={'/sign-up'}>Sign Up</Link>
                    </>
                    :
                    <>
                        <Image src={session.user.image! || ''} alt='logo' width={25} height={25} className='rounded-full'></Image>
                        <SignOutButton></SignOutButton>
                    </>
            }


        </div>
    )
}
```

```ts
// src/components/SignInForm.tsx

import { loginUser } from "@/actions/auth-actions";
import SocialButtons from "./SocialButtons";

export default function SignInForm() {
    return (
        <div className="max-w-md mx-auto space-y-5">
            <form action={loginUser} className="space-y-3">
                <input type="email" name="email" placeholder="Email" className="input input-bordered w-full" required />
                <input type="password" name="password" placeholder="Password" className="input input-bordered w-full" required />
                <button className="btn btn-primary w-full" type="submit">Sign In</button>
            </form>

            <SocialButtons />
        </div>
    );
}
```

```ts
// src/components/SignOutButton.tsx

import { doLogout } from '@/actions/auth-actions'
import React from 'react'

export default function SignOutButton() {
    return (
        <form action={doLogout}>
            <button className='btn btn-sm btn-error' type='submit' >Logout</button>
        </form>
    )
}
```

```ts
// src/components/SignUpForm.tsx

import { registerUser } from "@/actions/auth-actions";
import SocialButtons from "./SocialButtons";

export default function SignUpForm() {
    return (
        <div className="max-w-md mx-auto space-y-5">
            <form action={registerUser} className="space-y-3">
                <input type="text" name="name" placeholder="Name" className="input input-bordered w-full" required />
                <input type="url" name="image" placeholder="Image URL" className="input input-bordered w-full" />
                <input type="email" name="email" placeholder="Email" className="input input-bordered w-full" required />
                <input type="password" name="password" placeholder="Password" className="input input-bordered w-full" required />
                <button type="submit" className="btn btn-primary w-full">Sign Up</button>
            </form>

            <SocialButtons />
        </div>
    );
}
```

```ts
// src/components/SocialButtons.tsx

import { doSocialLogin } from '@/actions/auth-actions'
import React from 'react'

export default function SocialButtons() {
    return (
        <form action={doSocialLogin} className='text-center space-x-5'>
            <button className='btn' type='submit' name='action' value={"google"}>Sign In With Google</button>
            <button className='btn' type='submit' name='action' value={"github"}>Sign In With Github</button>
        </form>
    )
}
```

# Protect routes by middleware:  

```ts
// src/proxy.ts

import { NextResponse } from "next/server";
import { auth } from "@/auth";

const protectedRoutes = [
    "/products",
    "/checkout",
    "/orders",
];

const adminRoutes = [
    "/admin",
];

const sellerRoutes = [
    "/seller",
];

export default auth((req) => {

    const pathname = req.nextUrl.pathname;

    const isLoggedIn = !!req.auth;

    // logged-in users only
    const isProtectedRoute = protectedRoutes.some(route => pathname.startsWith(route));

    if (isProtectedRoute && !isLoggedIn) {

        return NextResponse.redirect(
            new URL("/sign-in", req.url)
        );
    }

    // admin only routes
    const isAdminRoute = adminRoutes.some(route => pathname.startsWith(route));

    if (isAdminRoute && req.auth?.user?.role !== "admin") {

        return NextResponse.redirect(
            new URL("/", req.url)
        );
    }

    // seller only routes
    const isSellerRoute = sellerRoutes.some(route => pathname.startsWith(route));

    if (isSellerRoute && req.auth?.user?.role !== "seller") {

        return NextResponse.redirect(
            new URL("/", req.url)
        );
    }

    return NextResponse.next();
});

export const config = {
    matcher: [
        "/products/:path*",
        "/checkout/:path*",
        "/orders/:path*",
        "/admin/:path*",
        "/secret/:path*",
        "/seller/:path*",
    ],
};
```

```ts
// src/app/admin/page.tsx

import { auth } from "@/auth";
import { redirect } from "next/navigation";

export default async function AdminPage() {

    const session = await auth();

    if (!session) {
        redirect("/sign-in");
    }

    if (session.user.role !== "admin") {
        redirect("/");
    }

    return (
        <div>
            Admin Page
        </div>
    );
}
```

```ts
// src/app/products/page.tsx

import { auth } from "@/auth";
import { redirect } from "next/navigation";

export default async function ProductsPage() {

    const session = await auth();

    if (!session) {
        redirect("/sign-in");
    }

    return (
        <div>
            Products Page
        </div>
    );
}
```

```ts
// src/app/seller/page.tsx

import { auth } from "@/auth";
import { redirect } from "next/navigation";

export default async function SecretPage() {

    const session = await auth();

    if (!session) {
        redirect("/sign-in");
    }

    if (session.user.role !== "seller") {
        redirect("/");
    }

    return (
        <div>
            Seller Page
        </div>
    );
}
```

# protect api: 

```ts
// src/app/api/admin/route.ts

import { auth } from "@/auth";

export async function GET() {

    const session = await auth();

    // not logged in
    if (!session) {

        return Response.json(
            {
                success: false,
                message: "Unauthorized",
            },
            {
                status: 401,
            }
        );
    }

    // not admin
    if (session.user.role !== "admin") {

        return Response.json(
            {
                success: false,
                message: "Forbidden",
            },
            {
                status: 403,
            }
        );
    }

    return Response.json({
        success: true,
        message: "Admin API",
        user: session.user,
    });
}
```

```ts
// src/app/api/seller/route.ts

import { auth } from "@/auth";

export async function GET() {

    const session = await auth();

    // not logged in
    if (!session) {

        return Response.json(
            {
                success: false,
                message: "Unauthorized",
            },
            {
                status: 401,
            }
        );
    }

    // not admin
    if (session.user.role !== "seller") {

        return Response.json(
            {
                success: false,
                message: "Forbidden",
            },
            {
                status: 403,
            }
        );
    }

    return Response.json({
        success: true,
        message: "Seller API",
        user: session.user,
    });
}
```

```ts
// src/app/api/user/route.ts

import { auth } from "@/auth";

export async function GET() {

    const session = await auth();

    if (!session) {

        return Response.json(
            {
                success: false,
                message: "Unauthorized",
            },
            {
                status: 401,
            }
        );
    }

    return Response.json({
        success: true,
        message: "Protected User API",
        user: session.user,
    });
}
```

```ts
// src/app/api/public/route.ts

export async function GET() {

    return Response.json({
        success: true,
        message: "Public API",
    });
}
```

# Example: 

https://github.com/tamim-111/next-auth-example
