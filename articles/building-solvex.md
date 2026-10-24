# From Expo Go to the Open Web: What I Learned Shipping SolveX

*By Sreeja Dende · October 2026 · 8 min read*

<p align="center">
  <img src="https://raw.githubusercontent.com/sreeja-dende/SolveX/main/screenshots/splash.jpeg" width="140" alt="SolveX logo"/>
</p>

**SolveX** is a platform where startups post real problems they haven't solved yet, and students compete to solve them. The startup picks the best answer, and the winner gets a certificate and points on a public leaderboard.

- **Live app:** https://solvex-web.onrender.com
- **Code:** https://github.com/sreeja-dende/SolveX

The app first ran only on my phone through Expo Go, talking to a server on my laptop over the same Wi-Fi. This article covers what it took to turn that into something anyone can open from a link: the bugs that appeared on the web, how I designed startup verification, and how I deployed the whole stack for free.

---

## Why I built it

Students learn from textbook problems with known answers. Startups, meanwhile, have real problems: an onboarding flow people abandon, a slow query, a pricing page nobody understands. A hackathon gives students a weekend and a made-up theme. I wanted something more ongoing: a startup posts one real problem, students take time to analyse it, and the startup gets several well-thought-out approaches.

So a SolveX submission isn't code. It's a structured analysis with five parts: **approach, root cause, proposed fix, implementation plan, and expected outcome.** That structure turned out to be the most important design decision. It makes submissions comparable, and it pushes students to think before proposing a solution.

## The architecture

```
React Native (Expo) app ──► Express REST API ──► MongoDB
   student / startup / admin      JWT auth, role checks     users, challenges,
   screens                        35 endpoints              solutions, documents
```

- **Frontend:** React Native with Expo SDK 54 and React Navigation. One codebase runs on Android, iOS and the web.
- **Backend:** Node.js and Express, with Mongoose models for users, challenges, solutions and verification documents.
- **Auth:** JWT tokens and bcrypt-hashed passwords. Every protected route checks the user's role (student, startup or admin) on the server, not only in the UI.

The core flow: a startup posts a challenge, students submit, the startup leaves feedback and star ratings, closes the challenge, and selects a winner. Selecting a winner awards the points, adds a certificate with a unique code to the student's profile, and updates the leaderboard in one step.

## Going from phone to browser: five bugs I didn't expect

Expo can export the same app to the web, so I assumed deploying would be a configuration task. It wasn't. These all worked perfectly on my phone and broke in the browser.

### 1. Forms that wouldn't scroll

On the web, the Create Account form stopped halfway down the screen and wouldn't scroll. The `ScrollView` was fine; the screen *around* it was the problem. On the web, the navigation library's screen container could grow as tall as its content, so the `ScrollView` inside never had a fixed height to scroll within. Giving every stack screen `cardStyle: { flex: 1 }` made it fill exactly the window, and every long form started scrolling.

**Lesson:** a scroll view only scrolls when something above it limits its height.

### 2. Inputs that lost focus after every letter

Typing a name in the registration form kept only the first letter. I had written the reusable `Field` input component *inside* the screen component. Every keystroke updated state, the screen re-rendered, React saw a brand-new `Field` component type, and it threw away the old input, focus included.

```js
// ❌ Re-created on every render, so React remounts the TextInput each keystroke
export default function RegisterScreen() {
  const Field = (props) => <TextInput {...props} />;
  ...
}

// ✅ Defined once at module level
function Field(props) { return <TextInput {...props} />; }
```

**Lesson:** never define components inside other components.

### 3. Confirmation dialogs that silently did nothing

"Close this challenge?" and "Select as winner?" use `Alert.alert` with buttons. On the web, `react-native-web` makes that call do nothing at all: no dialog, no error, and the action never runs. I wrote a small wrapper that uses `window.confirm` on the web and the native alert everywhere else, then replaced every import with it.

### 4. Lists that didn't refresh

After posting a challenge, the dashboard still said "0 challenges". Screens loaded their data once in `useEffect`, but tab screens stay mounted, so returning to a tab never re-fetched. Switching to `useFocusEffect` reloads the data every time a screen comes into view.

### 5. A server error from an empty star rating

Saving feedback without choosing a star rating crashed with a 500 error. The app sent `rating: 0`, and the database schema only allows 1–5. I fixed it on both sides: the app now omits an unset rating, and the server only stores a valid 1–5 value. The server should never trust the client to send clean data.

## Designing startup verification

Students need to trust that a startup is real before spending hours on its problem, so startups can earn a **Verified** badge by submitting company documents for an admin to review.

My first version only *pretended* to upload files. Building it properly raised three questions.

**Where do the files live?** My host's free tier wipes the server disk on every deploy, so saving uploads to disk would lose them. I store each document in MongoDB as binary data in its own collection, with a 2 MB limit per file, and exclude the file contents from every query except the one that serves the file.

**How do I stop someone uploading something harmful?** A file name and extension can be faked. The server reads the first bytes of each file (its *signature*: `%PDF`, the PNG header, `FF D8 FF` for JPEG) and rejects anything that isn't really a PDF or an image. This stops, for example, an HTML file renamed to `.pdf` from being served back to the admin.

**How does the admin open a private file in a new tab?** A browser tab can't send the admin's login token. So when the admin taps **Open**, the API returns a link signed with a short-lived token that's valid for 5 minutes and only for that one document. One more detail: the browser blocks `window.open` if it happens after an `await`, so the app opens a blank tab immediately on the click and points it at the link once the API responds.

The admin then approves or rejects. Rejection requires a reason, which the startup sees on its profile so it can fix the problem and resubmit.

## Deploying for ₹0

| Piece | Platform |
|---|---|
| Database | MongoDB Atlas free cluster (Mumbai region) |
| REST API | Render web service |
| Web app | Expo web export on a Render static site |

A few things I learned along the way:

- **Environment variables are baked in at build time.** The API address comes from `EXPO_PUBLIC_API_URL`, which Expo copies into the JavaScript bundle during the build. A cached build once kept an old address, so the build now always runs with `--clear`.
- **Lock down admin sign-up before going public.** Locally, anyone could register an admin through the API. On a public URL that's a serious hole. Creating an admin now requires a secret `ADMIN_SETUP_KEY` that exists only on the server.
- **Free servers sleep.** Render's free tier sleeps after 15 minutes without traffic, and waking takes about 50 seconds, which feels broken to a first-time visitor. A free scheduled ping every 10 minutes keeps the API awake. My first ping hit the health route, and the monitoring service marked it as failed because the response was too large to store. I added a `/api/ping` route that returns an empty `204 No Content`, and it's been green since.

All of the infrastructure is described in one `render.yaml` file, so the API and the web app deploy together from a single push.

## What I'd do differently

1. **Test on the web from day one.** Every bug above was cheap to fix but expensive to find late.
2. **Write the server's validation first.** The rating crash and the admin sign-up hole were both cases of the server trusting the client.
3. **Plan for where files live before writing upload code.** Knowing the disk would be wiped changed the whole design.

## What's next

- Push notifications when a new challenge is posted
- Profile pictures stored in cloud storage
- A scoring rubric so startups can compare submissions more consistently

If you're a student who wants real problems to work on, or a startup with one to share, try SolveX at **https://solvex-web.onrender.com**. The code is open at **https://github.com/sreeja-dende/SolveX**.

---

*Sreeja Dende is a final-year Computer Science student at RGUKT Basar, building full-stack web and mobile apps. [GitHub](https://github.com/sreeja-dende) · [LinkedIn](https://www.linkedin.com/in/sreeja-dende/)*
