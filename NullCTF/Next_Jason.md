Для начала давайте покажу структуру и внутренние файлы нашей задачи CTF. Буду ссылаться на эти файлы далее и использовать их как основу для нахождения уязвимости.  
1) /app/api/getFlag/route.js  
```js
import { NextResponse } from 'next/server';

export async function GET(req) {
	try {
		const token = req.cookies.get('token')?.value;
		const valid = await fetch(new URL('/token/verify', req.url), {
			method: 'POST',
			headers: {
				'Content-Type': 'application/json',
			},
			body: JSON.stringify({ token }),
		});
		const payload = await valid.json();
		if (!payload.valid) return NextResponse.json({ error: 'Invalid token or token missing' }, { status: 403 });
		if (payload.payload.username !== 'admin') return NextResponse.json({ error: 'You need to be admin!' }, { status: 403 });

		const flag = process.env.FLAG;

		return NextResponse.json({ flag });
	} catch (error) {
		console.error('Error retrieving public key:', error);
		return NextResponse.json({ error: 'Failed to retrieve public key' }, { status: 500 });
	}
}
```
2) /app/api/getPublicKey/route.js
```js
import { NextResponse } from 'next/server';
import { readFileSync } from 'fs';
import path from 'path';

const PUBKEY = readFileSync(path.join(process.cwd(), 'public.pem'), 'utf8');

export async function GET(req) {
	try {
		return NextResponse.json({ PUBKEY });
	} catch (error) {
		console.error('Error retrieving public key:', error);
		return NextResponse.json({ error: 'Failed to retrieve public key' }, { status: 500 });
	}
}
```
3) /app/api/login/route.js
```js
import { NextResponse } from 'next/server';

export async function POST(request) {
	try {
		const body = await request.json();
		const { username } = body;

		if (!username) {
			return NextResponse.json({ error: 'Username is required' }, { status: 400 });
		} else if (username === 'admin') {
			return NextResponse.json({ error: 'Try harder' }, { status: 403 });
		}

		const signResponse = await fetch(new URL('/token/sign', request.url), {
			method: 'POST',
			headers: {
				'Content-Type': 'application/json',
			},
			body: JSON.stringify({ username }),
		});

		if (!signResponse.ok) {
			throw new Error(`Token signing failed: ${signResponse.status}`);
		}

		const { token } = await signResponse.json();

		const response = NextResponse.json({ success: true });
		response.cookies.set({
			name: 'token',
			value: token,
			httpOnly: true,
		});

		return response;
	} catch (error) {
		console.error('Login error:', error);
		return NextResponse.json({ error: 'Authentication failed' }, { status: 500 });
	}
}
```
4) /app/token/sign/route.js
```js
import { NextResponse } from 'next/server';
import jwt from 'jsonwebtoken';
import { readFileSync } from 'fs';
import path from 'path';
const PRIVKEY = readFileSync(path.join(process.cwd(), 'private.pem'), 'utf8');

function signToken(payload) {
	return jwt.sign(payload, PRIVKEY, { algorithm: 'RS256' });
}

export async function POST(request) {
	try {
		const body = await request.json();

		if (!body || Object.keys(body).length === 0) {
			return NextResponse.json({ error: 'Payload required' }, { status: 400 });
		} else if (body.username === 'admin') {
			return NextResponse.json({ error: 'Try harder' }, { status: 403 });
		}

		const token = signToken(body);
		return NextResponse.json({ token });
	} catch (error) {
		console.error('Token signing error:', error);
		return NextResponse.json({ token: 'error', error: 'Failed to sign token' }, { status: 500 });
	}
}
```
5) /app/token/verify/route.js
```js
import { NextResponse } from 'next/server';
import jwt from 'jsonwebtoken';
import { readFileSync } from 'fs';
import path from 'path';
const PUBKEY = readFileSync(path.join(process.cwd(), 'public.pem'), 'utf8');

function verifyToken(token) {
	return jwt.verify(token, PUBKEY, { algorithms: ['RS256', 'HS256'] });
}

export async function POST(request) {
	try {
		const { token } = await request.json();

		if (!token) {
			return NextResponse.json({ error: 'Token required' }, { status: 400 });
		}

		const payload = verifyToken(token);
		return NextResponse.json({ valid: true, payload });
	} catch (error) {
		console.error('Token verification error:', error);
		return NextResponse.json({ valid: false, error: 'Invalid token' }, { status: 400 });
	}
}
```
Это все коды, которые могут нам потребоваться для решения и для нахождения уязвимости.
## Эксплуатация
---
Исходя из 1) мы видим - для вытаскивания флага нам нужен валидный токен для username = admin. В 5）мы видим，что токен получается с помощью **PUBKEY**  
Давайте начнем хотя бы с этой информации. Первое，что мы сделаем отправим запрос в caido в /token/sign  
<img width="1635" height="567" alt="image" src="https://github.com/user-attachments/assets/ba403c51-a5b2-47b7-b7f7-550e88b4910b" />    
Если мы введем ```"username"："admin"```， то мы получим **Try Harder**，получаем токен и идем с этим токеном на проверку в /token/verify  
<img width="1631" height="480" alt="image" src="https://github.com/user-attachments/assets/31e9da19-1ce0-4147-abfb-9bb307038244" />   
Отлично，токен валидный, можно попробовать получить PublicKey，а для этого вставим token в cookie и проверим，получим мы ключ или нет  
<img width="1630" height="570" alt="image" src="https://github.com/user-attachments/assets/416dc529-ec31-40f7-b4f4-c35f2b770d3f" />  
Замечательно, мы забрали PublicKey и теперь мы можем с помощью него постараться подделать наш jwt токен, но для начала проверим его на ```https://10015.io/tools/jwt-encoder-decoder```  
<img width="1411" height="628" alt="image" src="https://github.com/user-attachments/assets/8032f9de-9877-4b32-b150-1bd5a5a5822f" /> 
Мы видим，что наш jwt token с '''username = admin1''' и теперь пробуем зашифровать токен с '''username = admin''' с помощью нашего **Public Key** и обязательно выбираем алгоритм '''HS256'.Пробуем)  
<img width="1555" height="729" alt="image" src="https://github.com/user-attachments/assets/e0ef0a95-3ac2-4c80-beb1-1ab77e3e850d" />   
Мы получили наш токен！Теперь пробуем доступ к /api/getFlag，а в куки вставляем наш зашифрованный токен.  
<img width="1633" height="522" alt="image" src="https://github.com/user-attachments/assets/629f062b-fea9-455d-8143-613b25e31ac4" />      
Мы получили флаг!  
```json
{
    "flag": "nullctf{f0rg3_7h15_cv3_h3h_9b82faf38b63811a}"
}
```
