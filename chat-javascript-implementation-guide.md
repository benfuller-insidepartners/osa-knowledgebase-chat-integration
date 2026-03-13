# One Step After — Webchat Widget Setup

## 1. Import & Activate the Workflow

1. Import `n8n-webchat-workflow.json` into n8n Cloud
2. Click on the **Chat Trigger** node
3. Copy the **Production URL** — it will look something like:
   ```
   https://onestepafter.app.n8n.cloud/webhook/XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX/chat
   ```
4. Toggle the workflow to **Active**

## 2. Embed on Your Website

Drop this snippet into any page, just before the closing `</body>` tag. Replace the `webhookUrl` with your actual production URL from step 3.

```html
<!-- One Step After Chat Widget -->
<link href="https://cdn.jsdelivr.net/npm/@n8n/chat/dist/style.css" rel="stylesheet" />
<script type="module">
  import { createChat } from 'https://cdn.jsdelivr.net/npm/@n8n/chat/dist/chat.bundle.es.js';

  createChat({
    webhookUrl: 'https://onestepafter.app.n8n.cloud/webhook/XXXXXXXX/chat',
    mode: 'window',
    chatInputKey: 'chatInput',
    metadata: {},
    showWelcomeScreen: true,
    initialMessages: [
      'Hi! I\'m here to help with estate planning, probate, and the steps involved after someone passes away.',
      'You can ask me about wills, trusts, powers of attorney, probate thresholds, executor responsibilities, and more.'
    ],
    i18n: {
      en: {
        title: 'One Step After',
        subtitle: 'Estate planning & execution assistant',
        getStarted: 'Start a Conversation',
        inputPlaceholder: 'Ask about estate planning, probate, wills...',
        footer: ''
      }
    },
    theme: {
      button: {
        backgroundColor: '#1a365d',
        iconColor: '#ffffff',
        right: 20,
        bottom: 20,
        size: 'medium'
      },
      chatWindow: {
        backgroundColor: '#ffffff',
        textColor: '#1a202c',
        title: {
          backgroundColor: '#1a365d',
          textColor: '#ffffff'
        },
        input: {
          backgroundColor: '#f7fafc',
          textColor: '#1a202c',
          borderColor: '#e2e8f0'
        },
        userMessage: {
          backgroundColor: '#1a365d',
          textColor: '#ffffff'
        },
        botMessage: {
          backgroundColor: '#f7fafc',
          textColor: '#1a202c'
        }
      }
    }
  });
</script>
```

## 3. Bubble Integration

For Bubble specifically, add the snippet via:

1. Go to **Settings → SEO/metatags → Script/meta tags in the body**
2. Paste the snippet above
3. This loads the chat widget on every page

Alternatively, to show it only on specific pages:
1. Add an **HTML element** on the target page
2. Paste the snippet inside it

## 4. Customization

### Colors

Update the `theme` object to match One Step After's branding:

```javascript
button: {
  backgroundColor: '#1a365d',  // Button background
  iconColor: '#ffffff',         // Button icon color
},
chatWindow: {
  title: {
    backgroundColor: '#1a365d',  // Header background
    textColor: '#ffffff'          // Header text
  },
  userMessage: {
    backgroundColor: '#1a365d',  // User message bubbles
    textColor: '#ffffff'
  },
  botMessage: {
    backgroundColor: '#f7fafc',  // Assistant message bubbles
    textColor: '#1a202c'
  }
}
```

### Position

Change where the chat button appears:

```javascript
button: {
  right: 20,   // pixels from right edge
  bottom: 20,  // pixels from bottom edge
  size: 'medium'  // 'small', 'medium', or 'large'
}
```

### Window vs Fullscreen

```javascript
mode: 'window'     // Floating chat window (default)
mode: 'fullscreen'  // Takes over the full page
```

## 5. How It Differs from the Webhook API

| Feature | Webhook API | Webchat Widget |
|---------|-------------|----------------|
| Frontend | Custom (Bubble builds it) | Pre-built n8n widget |
| Conversation history | Client-managed, sent per request | Per-session (no cross-session memory) |
| Auth | API key header | None (public widget) |
| State filtering | Via `metadata.userState` | Not available (defaults to ALL) |
| User metadata | Full control | Limited |
| Styling | Fully custom | Configurable via theme |
| Response format | JSON with sourceDocs | Rendered Markdown in chat bubble |

The webchat widget is great for a quick deployment or a public-facing help widget. The webhook API gives the Bubble developer full control over the UI and user context.

## 6. Restricting Access

To limit which domains can use the widget, update `allowedOrigins` in the Chat Trigger node:

```
onestepafter.com,*.onestepafter.com
```

This prevents other websites from embedding your chat widget.
