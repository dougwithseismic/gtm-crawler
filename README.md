# GTM Crawler

A local-first crawler for Go To Market teams with big ambitions. Built with TypeScript, Express, and Playwright, this tool allows you to crawl websites and extract valuable data for your GTM strategy. Features include plugin-based analysis, webhook notifications, and seamless integration with Clay.com and n8n.

## Table of Contents

- [Installation](#installation)
- [Tunnel Setup](#tunnel-setup)
- [Quick Start](#quick-start)
- [Using with n8n](#using-with-n8n)
- [Using with Clay.com](#using-with-claycom)
- [Webhook Integration](#webhook-integration)
- [Writing Custom Plugins](#writing-custom-plugins)
- [API Reference](#api-reference)

## Installation

1. Clone the repository:

```bash
git clone https://github.com/dougwithseismic/gtm-crawler.git
cd gtm-crawler
```

2. Install dependencies:

```bash
pnpm install
```

## Tunnel Setup

The crawler requires a public URL for webhook notifications. We use ngrok for this:

1. Run the tunnel setup script:

```bash
pnpm setup:tunnel
```

2. Follow the prompts:
   - If you don't have an ngrok account, the script will guide you to create one
   - Copy your ngrok auth token from <https://dashboard.ngrok.com/get-started/your-authtoken>
   - The script will install ngrok if needed and configure it with your token

3. Save your tunnel URL:
   - The script will create a tunnel and show you the public URL
   - Save this URL - you'll need it for webhook configurations
   - Example: `https://your-subdomain.ngrok.io`

## Quick Start

1. Start the service:

```bash
pnpm dev
```

The service will automatically create a tunnel and display the public URL in the console.

2. Test the crawler with the built-in test webhook:

```bash
# Replace YOUR_TUNNEL_URL with the URL shown in your console
curl -X POST "YOUR_TUNNEL_URL/crawl/example.com" \
  -H "Content-Type: application/json" \
  -d '{
    "webhook": {
      "url": "YOUR_TUNNEL_URL/test/webhook"
    }
  }'
```

3. Watch the webhook responses in your console

## Using with n8n

1. Create a new workflow in n8n
2. Add an HTTP Request node:
   - Method: POST
   - URL: Your tunnel URL + /crawl/[target-domain]
   - Body:

   ```json
   {
     "maxDepth": 3,
     "maxPages": 100,
     "webhook": {
       "url": "https://your-n8n-webhook-url",
       "on": ["completed"]  // Only get final results
     }
   }
   ```

3. Add a Webhook node to receive results:
   - Authentication: None (or add headers in the crawler request)
   - Method: POST
   - Path: /crawler-webhook
   - Response Code: 200
   - Response Data: JSON

4. Use the "Function" node to process results:

```javascript
return {
  json: {
    processedData: items[0].json.result.summary,
    // Add your processing logic here
  }
}
```

## Using with Clay.com

1. Create a new Clay table
2. Add a "Webhook" column
3. In your Clay workflow:

```javascript
// Clay.com workflow example
const response = await fetch('https://your-tunnel-url/crawl/' + domain, {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    maxPages: 100,
    webhook: {
      url: clayWebhookUrl,
      headers: { 'X-Clay-Token': 'your-token' },
      on: ['completed']
    }
  })
});

// The crawler will send results to your Clay webhook
```

## Webhook Integration

The crawler supports flexible webhook configurations:

```typescript
interface WebhookConfig {
  url: string;
  headers?: Record<string, string>;
  retries?: number;
  on?: ('started' | 'progress' | 'completed' | 'failed')[];
}
```

Example webhook payloads:

1. Job Started:

```json
{
  "status": "started",
  "jobId": "job-uuid",
  "timestamp": "2024-01-03T12:00:00.000Z"
}
```

2. Progress Update:

```json
{
  "status": "progress",
  "jobId": "job-uuid",
  "timestamp": "2024-01-03T12:00:10.000Z",
  "progress": {
    "pagesAnalyzed": 50,
    "totalPages": 100,
    "currentUrl": "https://example.com/page"
  }
}
```

3. Job Completed:

```json
{
  "status": "completed",
  "jobId": "job-uuid",
  "timestamp": "2024-01-03T12:01:00.000Z",
  "result": {
    "pages": [...],
    "summary": {...}
  }
}
```

## Writing Custom Plugins

Create a new plugin by implementing the `CrawlerPlugin` interface:

1. Create a new file in `src/services/crawler/plugins/`:

```typescript
import { CrawlerPlugin, Page } from '../types/plugin';
import type { CrawlJob } from '../types.improved';

interface MyMetrics {
  customValue: number;
}

interface MySummary {
  totalCustomValue: number;
}

export class MyPlugin implements CrawlerPlugin<'myPlugin', MyMetrics, MySummary> {
  readonly name = 'myPlugin';
  enabled = true;

  // Called once when the crawler service starts
  async initialize(): Promise<void> {
    // Set up plugin resources
  }

  // Called before starting to crawl a website
  async beforeCrawl(job: CrawlJob): Promise<void> {
    console.log(`Starting crawl of ${job.config.url}`);
  }

  // Called before each page is loaded
  async beforeEach(page: Page): Promise<void> {
    // Add custom scripts or prepare page
    await page.addScriptTag({
      content: `window.__myPlugin = { startTime: Date.now() };`,
    });
  }

  // Main page analysis
  async evaluate(page: Page, loadTime: number): Promise<Record<'myPlugin', MyMetrics>> {
    const metrics = await page.evaluate(() => ({
      customValue: document.querySelectorAll('.my-selector').length,
    }));
    return { myPlugin: metrics };
  }

  // Called after page analysis is complete
  async afterEach(page: Page): Promise<void> {
    // Clean up page modifications
    await page.evaluate(() => {
      delete (window as any).__myPlugin;
    });
  }

  // Called after completing a website crawl
  async afterCrawl(job: CrawlJob): Promise<void> {
    console.log(`Completed crawl of ${job.config.url}`);
  }

  // Aggregate results from all pages
  async summarize(pages: Array<Record<'myPlugin', MyMetrics>>): Promise<Record<'myPlugin', MySummary>> {
    return {
      myPlugin: {
        totalCustomValue: pages.reduce((sum, page) => sum + page.myPlugin.customValue, 0),
      },
    };
  }

  // Called when the crawler service shuts down
  async destroy(): Promise<void> {
    // Clean up plugin resources
  }
}
```

2. Register the plugin with the crawler:

```typescript
import { MyPlugin } from './plugins/my-plugin';

const crawler = new CrawlerService({
  plugins: [new MyPlugin()],
  config: { debug: true }
});
```

### Plugin Lifecycle

The plugin system provides several hooks that are called at different stages:

1. **Service Lifecycle**
   - `initialize()`: Called when the crawler service starts
   - `destroy()`: Called when the crawler service shuts down

2. **Job Lifecycle**
   - `beforeCrawl(job)`: Called before starting a website crawl
   - `afterCrawl(job)`: Called after completing a website crawl

3. **Page Lifecycle**
   - `beforeEach(page)`: Called before loading each page
   - `evaluate(page, loadTime)`: Called to analyze each page
   - `afterEach(page)`: Called after analyzing each page

4. **Results Processing**
   - `summarize(pages)`: Called to generate final analysis

All hooks except `evaluate` and `summarize` are optional. Use them to:

- Set up and clean up resources
- Add custom scripts to pages
- Track crawling progress
- Collect additional metrics
- Clean up page modifications

## API Reference

See the [API Documentation](./docs/api-reference.md) for detailed endpoint specifications and configuration options.

## License

MIT
