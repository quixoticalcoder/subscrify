# subscrify frontend

The React client for subscrify contains the staff workspace, customer storefront, authentication screens, and optional Groq assistant. See the [root README](../README.md) for backend setup, demo accounts, workflows, and implementation limits.

## Run locally

Use Node.js 20.19+ or 22.12+ and start the Flask API first.

```bash
npm ci
cp .env.example .env
npm run dev
```

`VITE_BASE_URL` is the Flask origin, normally `http://localhost:5000`. Do not append `/api`; the Axios wrappers already include it. Vite reads this configuration at startup/build time.

## Commands

| Command | Purpose |
| --- | --- |
| `npm run dev` | Start Vite development server. |
| `npm run build` | Compile production assets to `dist/`. |
| `npm run preview` | Preview the build locally. |
| `npm run lint` | Run ESLint; currently reports existing errors and warnings. |

## Code map

- `src/App.jsx`: route definitions and client guards.
- `src/layout/`: staff and customer navigation shells.
- `src/pages/`: authentication, admin, and portal screens.
- `src/configs/api.js`: API wrappers and JWT request/401 response handling.
- `src/store/`: persisted auth, cart, and theme state.
- `src/components/common/`: shared controls, tours, and the Groq assistant.

The application uses React 19, React Router 7, Redux Toolkit, Tailwind CSS 4, Recharts, and GSAP. Browser persistence uses `subscrify_` keys. A 401 response clears the stored session and returns the user to login.

The optional assistant saves its Groq key in local storage and sends application context directly to Groq. Server credentials must never be placed in frontend environment variables. Client route guards do not replace API authorization.

For hosting, build with the correct API URL and configure nested routes to fall back to `index.html`. `npm run preview` is intended for local build inspection.
