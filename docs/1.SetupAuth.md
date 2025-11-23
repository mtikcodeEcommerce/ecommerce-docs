# Setup Authorization

## Install

- Better Auth: `npm install better-auth`
- Nodemailer: `npm install modemailer`

## Create Google Auth Platform

- Set Authorized redirect URLs is `ecommerce-backend` API endpoint
![Google Cloud Auth Platform Setup Example](https://mtikcodeecommerce.github.io/ecommerce-docs/img/google-auth-platform.png)

## Setup Configuration

- Setup `src/app/auth/auth.config.ts`:
```ts
import { betterAuth } from 'better-auth';
import { Pool } from 'pg';
import * as dotenv from 'dotenv';
import { EmailService } from '../email/email.service';
import { ConfigService } from '@nestjs/config';

dotenv.config();

const configService = new ConfigService();
const emailService = new EmailService(configService);

const pool = new Pool({
    host: process.env.DB_HOST || 'localhost',
    port: parseInt(process.env.DB_PORT, 10) || 5432,
    user: process.env.DB_USERNAME || 'postgres',
    password: process.env.DB_PASSWORD || 'password',
    database: process.env.DB_NAME || 'ecommerce',
    ssl: {
        rejectUnauthorized: false,
    },
    max: 20,
    idleTimeoutMillis: 30000,
    connectionTimeoutMillis: 2000,
});

export const auth = betterAuth({
    database: pool,

    emailAndPassword: {
        // Enable email and password authentication method
        // When true, users can sign up/sign in with email and password
        enabled: true,
        
        // Require users to verify their email before they can fully access the app
        // When true, users must click a verification link sent to their email
        requireEmailVerification: true,
        
        // Function called when a user requests a password reset
        // This is an async callback that Better Auth will invoke
        sendResetPassword: async ({ user, url, token }) => {
            // Log the reset token to console (useful for debugging)
            // In production, remove this or use proper logging
            console.log('tokenResetPassword', token);
            
            // Call your custom EmailService to send the password reset email
            // Parameters:
            // - user.email: recipient's email address
            // - user.name: recipient's name for personalization
            // - url: the complete URL the user should click to reset password
            await emailService.sendPasswordResetEmail(
                user.email,
                user.name,
                url,
            );
        },
        
        // Function called after a user successfully resets their password
        // Useful for logging, analytics, or triggering other actions
        onPasswordReset: async ({ user }) => {
            // Log the password reset event
            // In production, you might want to:
            // - Send a notification email
            // - Log to a security audit trail
            // - Invalidate all existing sessions
            console.log(`Password for user ${user.email} has been reset.`);
        },
    },

    // ========================================================================
    // EMAIL VERIFICATION CONFIGURATION
    // ========================================================================
    
    emailVerification: {
        // Automatically send verification email when a new user signs up
        // When true, email is sent immediately after registration
        sendOnSignUp: true,
        
        // Whether to automatically sign in the user after email verification
        // false = user must sign in manually after verifying email
        // true = user is automatically signed in after clicking verification link
        autoSignInAfterVerification: false,
        
        // How long the verification token is valid (in seconds)
        // 24 * 60 * 60 = 86400 seconds = 24 hours
        // After this time, the verification link will expire
        expiresIn: 24 * 60 * 60,
        
        // Function called to send the verification email
        // This is where you implement your custom email sending logic
        sendVerificationEmail: async ({ user, token }) => {
            // Log the verification token (useful for debugging/testing)
            // In development, you can copy this token to manually verify emails
            console.log('tokenVerificationEmail', token);
            
            // Construct the verification URL
            // Users will click this link to verify their email address
            // Format: https://your-domain.com/auth/verify-email?token=ABC123&email=user@example.com
            const verificationUrl = `${process.env.BETTER_AUTH_URL}/auth/verify-email?token=${token}&email=${user.email}`;
            
            // Call your custom EmailService to send the verification email
            // The email should contain the verificationUrl as a clickable link
            await emailService.sendVerificationEmail(
                user.email,      // Recipient's email address
                user.name,       // Recipient's name for personalization
                verificationUrl, // The URL to verify the email
            );
        },
    },

    // ========================================================================
    // SOCIAL AUTHENTICATION PROVIDERS (OAuth)
    // ========================================================================
    
    socialProviders: {
        // Google OAuth configuration
        // Allows users to sign in with their Google account
        google: {
            // Client ID from Google Cloud Console
            // Get this by creating a project at https://console.cloud.google.com
            // Falls back to empty string (disables Google login if not set)
            clientId: process.env.GOOGLE_CLIENT_ID || '',
            
            // Client Secret from Google Cloud Console
            // This is a secret key - NEVER commit it to version control!
            clientSecret: process.env.GOOGLE_CLIENT_SECRET || '',
        },
    },

    // ========================================================================
    // SECURITY & SERVER CONFIGURATION
    // ========================================================================
    
    // Secret key used to sign and verify tokens (JWT, sessions, etc.)
    // Generate one using: node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
    secret: process.env.BETTER_AUTH_SECRET || '',
    
    // Base URL of your backend application
    // This is where Better Auth API endpoints will be hosted
    // Example: https://api.yourdomain.com or http://localhost:8080
    baseURL: process.env.BETTER_AUTH_URL || 'http://localhost:8080',
    
    // Base path for all Better Auth endpoints
    // All auth endpoints will be under this path
    // Example: /api/auth/signin, /api/auth/signup, /api/auth/signout
    basePath: '/api/auth',

    // ========================================================================
    // CORS & TRUSTED ORIGINS
    // ========================================================================
    
    // Array of trusted origins that can make requests to your auth endpoints
    // This is important for CORS (Cross-Origin Resource Sharing) security
    trustedOrigins: [
        // Your backend URL - where the auth server is hosted
        process.env.BETTER_AUTH_URL || 'http://localhost:8080',
        
        // Your frontend URL - where your web app is hosted
        // Requests from this origin will be allowed
        process.env.CLIENT_URL || 'http://localhost:3000',
        
        // Add more origins if you have multiple frontends:
        // 'https://admin.yourdomain.com',
        // 'https://mobile.yourdomain.com',
    ],

    // ========================================================================
    // SESSION CONFIGURATION
    // ========================================================================
    
    session: {
        // Name of the cookie that stores the session token
        // This cookie will be set in the user's browser
        cookieName: 'better-auth.session_token',
        
        // How long a session lasts before it expires (in seconds)
        // After this time, the user will need to sign in again
        expiresIn: 60 * 60 * 24 * 7,
        
        // How often to update the session expiry time (in seconds)
        // This implements a "sliding window" session - active users stay logged in
        updateAge: 60 * 60 * 24,
    },
});

export type Auth = typeof auth;
```

- Setup `src/app/email/email.service.ts`
```ts
import { Injectable, Logger } from '@nestjs/common';
import * as nodemailer from 'nodemailer';
import { ConfigService } from '@nestjs/config';
import * as dotenv from 'dotenv';

dotenv.config();

export interface EmailOptions {
    to: string | string[];
    subject: string;
    text?: string;
    html?: string;
    from?: string;
}

@Injectable()
export class EmailService {
    private readonly logger = new Logger(EmailService.name);
    private transporter: nodemailer.Transporter;

    constructor(private configService: ConfigService) {
        this.createTransporter();
    }

    private createTransporter() {
        this.transporter = nodemailer.createTransport({
            service: 'gmail',
            auth: {
                user: process.env.SMTP_USER,
                pass: process.env.SMTP_PASS,
            },
        });

        void this.verifyConnection();
    }

    private async verifyConnection() {
        try {
            await this.transporter.verify();
            this.logger.log('Kết nối SMTP thành công');
        } catch (error) {
            this.logger.error('Lỗi kết nối SMTP:', error);
        }
    }

    async sendEmail(options: EmailOptions): Promise<boolean> {
        try {
            const mailOptions = {
                from:
                    options.from ||
                    process.env.SMTP_FROM ||
                    'dev.bxmt@projgis.link',
                to: Array.isArray(options.to)
                    ? options.to.join(', ')
                    : options.to,
                subject: options.subject,
                text: options.text,
                html: options.html,
            };

            const info = await this.transporter.sendMail(mailOptions);
            this.logger.log(`Email đã được gửi thành công: ${info.messageId}`);
            return true;
        } catch (error) {
            this.logger.error('Lỗi gửi email:', error);
            return false;
        }
    }

    async sendPasswordResetEmail(
        to: string,
        name: string,
        url: string,
    ): Promise<boolean> {
        const subject = 'Đặt lại mật khẩu';

        const html = `
      <div style="font-family: Arial, sans-serif; max-width: 600px; margin: 0 auto;">
        <h2 style="color: #333;">Đặt lại mật khẩu</h2>
        <p>Xin chào ${name},</p>
        <p>Chúng tôi nhận được yêu cầu đặt lại mật khẩu cho tài khoản của bạn.</p>

        <div style="background-color: #f4f4f4; padding: 20px; margin: 20px 0; border-radius: 5px; text-align: center;">
          <a href="${url}"
             style="background-color: #007bff; color: white; padding: 12px 24px; text-decoration: none; border-radius: 5px; display: inline-block;">
            Đặt lại mật khẩu
          </a>
        </div>

        <p>Nếu bạn không yêu cầu đặt lại mật khẩu, vui lòng bỏ qua email này.</p>
        <p><strong>Lưu ý:</strong> Link này sẽ hết hạn sau 1 giờ.</p>

        <p>Trân trọng,<br>Đội ngũ Ecommerce Store</p>
      </div>
    `;

        return this.sendEmail({
            to,
            subject,
            html,
        });
    }

    async sendVerificationEmail(
        to: string,
        name: string,
        url: string,
    ): Promise<boolean> {
        const subject = 'Xác thực email';

        const html = `
      <div style="font-family: Arial, sans-serif; max-width: 600px; margin: 0 auto;">
        <h2 style="color: #333;">Xác thực email</h2>
        <p>Xin chào ${name},</p>
        <p>Cảm ơn bạn đã đăng ký tài khoản! Vui lòng xác thực email để hoàn tất quá trình đăng ký.</p>

        <div style="background-color: #f4f4f4; padding: 20px; margin: 20px 0; border-radius: 5px; text-align: center;">
          <a href="${url}"
             style="background-color: #28a745; color: white; padding: 12px 24px; text-decoration: none; border-radius: 5px; display: inline-block;">
            Xác thực email
          </a>
        </div>

        <p>Nếu bạn không tạo tài khoản này, vui lòng bỏ qua email này.</p>

        <p>Trân trọng,<br>Đội ngũ Ecommerce Store</p>
      </div>
    `;

        return this.sendEmail({
            to,
            subject,
            html,
        });
    }
}
```

- Setup `.env`:
```.env
DB_HOST=
DB_PORT=5434
DB_USERNAME=
DB_PASSWORD=
DB_NAME=

# Google OAuth
// If user's emai has 2FA google authorization, see: https://nodemailer.com/usage/using-gmail
SMTP_USER=
SMTP_PASS=
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=

# Better Auth
BETTER_AUTH_SECRET=
BETTER_AUTH_URL=
CLIENT_URL=
```
![Google Authorization 2FA config](https://mtikcodeecommerce.github.io/ecommerce-docs/img/google-2fa-config.png)
