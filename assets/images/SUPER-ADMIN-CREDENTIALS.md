# SUPER Admin credentials: how to locate or bootstrap (staging/prod)

This guide explains how actuator access is secured, how to find which account holds the SUPER role, and what to do if you need to bootstrap or reset a SUPER admin in staging/prod.

Important principles
- Actuator endpoints are protected by default. Only health/info are public; all other actuator endpoints require authentication (JWT). A browser may show a login prompt if it sees a 401; use a Bearer token instead of Basic auth.
- SUPER admin passwords are never stored in plaintext. You cannot “look up” a password; you can only authenticate as the user, reset the password, or promote a known user to SUPER under a controlled procedure.

1) Actuator access model (api-gateway and services)
- Public: /actuator/health, /actuator/health/**, and GET /actuator/info
- Protected: all other /actuator/** endpoints require authentication
- How to call with a JWT (example):
  1. Obtain a JWT by authenticating against the authentication service.
     curl -s -X POST "https://api.staging.ujomorserver.com/api/v1/authentication/authenticate" \
       -H "Content-Type: application/json" \
       -d '{"email":"<admin-email>","password":"<admin-password>"}' | jq -r .jwtAccessToken
  2. Use the token to call actuator:
     TOKEN="<value-from-step-1>"
     curl -H "Authorization: Bearer $TOKEN" https://api.staging.ujomorserver.com/actuator

Note: The “login prompt” you see in a browser is the default browser reaction to 401; our services use Bearer JWT, not Basic. Prefer curl or API tools with Authorization headers.

2) Find the SUPER admin account (RDS)
Users are stored in the authentication database (MariaDB). The user table is named User (case-sensitive depending on collation). The role is stored in column user_role (enum), e.g., SUPER, ADMIN2, USER.

SQL (read-only) to list SUPER admins:
SELECT id, email, user_role, account_is_active, last_logged_in
FROM User
WHERE user_role = 'SUPER';

If you do not know how to connect to RDS in staging/prod, follow your team’s standard: either through a bastion host, AWS Systems Manager port forwarding, or a private EC2 jumpbox. Never expose RDS publicly.

3) If no SUPER exists in staging: bootstrap one (staging only)
Prefer the lightest, auditable path.
A) Register a normal account via API (gets USER role):
   curl -s -X POST https://api.staging.ujomorserver.com/api/v1/authentication/register \
     -H "Content-Type: application/json" \
     -d '{"email":"you@example.com","password":"<StrongPassword12!>"}'
B) In RDS, promote that user to SUPER and activate it:
   UPDATE User SET user_role = 'SUPER', account_is_active = TRUE WHERE email = 'you@example.com';
C) Immediately rotate the password: authenticate and change it per policy.

Operational notes
- Perform the SQL change using a change request, in staging only, and log the approver and timestamp.
- Rotate credentials after promotion. Enforce a strong password per policy (min length, history).
- Consider adding a ticket to provide an admin-only endpoint for role elevation guarded by break-glass controls if frequent.

4) If you need to reset a known SUPER admin’s password
- There is no public "forgot password" flow wired here. Use the database-admin path:
  1) Set account_is_active = TRUE for the account if locked.
  2) Either:
     - Temporarily set a strong one-time password hash in the password column (requires generating a bcrypt hash offline consistent with Spring Security), or
     - Promote a second trusted account to SUPER (as in 3B) and use it going forward; demote/disable the old account if needed.
- Afterward, force a password rotation and verify recent tokens are revoked (the system revokes on login flows).

5) Where are SUPER admin credentials stored?
- Email/role live in the authentication service database (RDS). Passwords are stored only as hashes (bcrypt), not recoverable.
- There is no Secrets Manager entry for a SUPER password by design.

6) Quick checklist
- Need to see actuator? Use a JWT and Authorization: Bearer <token> header.
- Need a SUPER in staging? Register an account, then UPDATE User.user_role='SUPER'.
- Need to know who the SUPER is? Query: SELECT email FROM User WHERE user_role='SUPER'.
- Never store plaintext passwords. Prefer rotation and role-based access.

Appendix: sample AWS RDS connect (Systems Manager port forward)
- Start a port-forwarding session to the RDS endpoint via an SSM-managed instance:
  aws ssm start-session \
    --target <INSTANCE_ID> \
    --document-name AWS-StartPortForwardingSessionToRemoteHost \
    --parameters '{"host":["<RDS_ENDPOINT>"],"portNumber":["3306"],"localPortNumber":["3306"]}'
- Then connect locally: mysql -h 127.0.0.1 -P 3306 -u <user> -p

Security reminder
- Treat any direct DB role change as a break-glass procedure. Limit to staging or emergency in prod with approvals, and revert to standard processes when possible.
