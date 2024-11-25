First, I will give you some potentially helpful context about my code.  
Then, I will show you the insertion point and give you the instruction. The insertion point will be in `src/index.ts`.  

-------  

## Potentially helpful context  

#### file_context_4  

`package.json` from line 1:  
```json  
{  
  \"name\": \"openai-caching-proxy-worker\",  
  \"version\": \"0.0.0\",  
  \"devDependencies\": {  
    \"@cloudflare/workers-types\": \"^4.20221111.1\",  
    \"@types/jest\": \"^29.2.4\",  
    \"@typescript-eslint/eslint-plugin\": \"^5.46.0\",  
    \"@typescript-eslint/parser\": \"^5.46.0\",  
    \"eslint\": \"^8.29.0\",  
    \"eslint-config-prettier\": \"^8.5.0\",  
    \"eslint-import-resolver-typescript\": \"^3.5.2\",  
    \"eslint-plugin-eslint-comments\": \"^3.2.0\",  
    \"eslint-plugin-import\": \"^2.26.0\",  
    \"eslint-plugin-jest\": \"^27.1.6\",  
    \"eslint-plugin-json\": \"^3.1.0\",  
    \"eslint-plugin-no-async-without-await\": \"^1.2.0\",  
    \"eslint-plugin-prettier\": \"^4.2.1\",  
    \"jest\": \"^29.3.1\",  
    \"jest-environment-miniflare\": \"^2.11.0\",  
    \"prettier\": \"^2.8.1\",  
    \"ts-jest\": \"^29.0.3\",  
    \"ts-node\": \"^10.9.1\",  
    \"tsconfig-paths\": \"^4.1.1\",  
    \"tsconfig-paths-jest\": \"^0.0.1\",  
    \"typescript\": \"^4.9.4\",  
    \"wrangler\": \"^3.88.0\",  
    \"yarn-deduplicate\": \"^6.0.0\"  
  },  
  \"private\": true,  
  \"scripts\": {  
    \"start\": \"wrangler dev\",  
    \"deploy\": \"wrangler deploy\",  
    \"check-types\": \"tsc --noEmit\",  
    \"deduplicate\": \"yarn-deduplicate && yarn install\",  
    \"lint\": \"yarn lint:ts --cache\",  
    \"lint:ci\": \"yarn lint:ts\",  
    \"lint:ts\": \"eslint . --ext .js,.ts,.json\",  
    \"prettier:all\": \"prettier --write \\\"**/*.+(js|jsx|ts|tsx|json|css|html)\\\"\",  
    \"test\": \"jest --config test/jest.config.js\"  
  },  
  \"dependencies\": {  
    \"@upstash/redis\": \"^1.18.1\"  
  }  
}  
```  

#### file_context_3  

`wrangler.toml` from line 1:  
```toml  
name = \"openai-caching-proxy-worker\"  
main = \"src/index.ts\"  
compatibility_date = \"2022-12-11\"  

# This cloudflare worker uses either Upstash or workers KV to store cache.  

# To use Upstash (hosted redis-over-REST),  
# https://docs.upstash.com/redis/quickstarts/cloudflareworkers  
#  
# 1. Signing up for a free account at upstash.com, set your secrets below based on instructions  
#    here for `wrangler secret put` (or alternatively use the Cloudflare Workers web UI):  
#    https://developers.cloudflare.com/workers/platform/environment-variables/#adding-secrets-via-wrangler  
# 2. Uncomment the following environment variables config:  
# [vars]  
# - UPSTASH_REDIS_REST_TOKEN  
# - UPSTASH_REDIS_REST_URL  

# To use workers KV,  
# 1. Run `wrangler kv:namespace create <YOUR_NAMESPACE>` in your terminal  
# 2. Uncomment the following config and fill in <YOUR_ID>:  
# kv_namespaces = [  
#    { binding = \"OPENAI_CACHE\", id = \"<YOUR_ID>\" }  
# ]  

[observability.logs]  
enabled = true  
```  

#### file_context_1  

`src/proxy.ts` from line 1:  
```typescript  
import { getCacheKey, ResponseCache } from './cache';  
import { Env } from './env';  
import { getHeadersAsObject } from './utils';  

interface HandleProxyOpts {  
  request: Request;  
  env: Env;  
  ctx: ExecutionContext;  
  ttl: number | null;  
  pathname: string;  
}  

export const handleProxy = async ({  
  request,  
  env,  
  ctx,  
  ttl,  
  pathname,  
}: HandleProxyOpts): Promise<Response> => {  
  const fetchMethod = request.method;  
  const fetchPath = pathname.replace(/^\\/proxy/, '');  
  const fetchUrl = `https://api.openai.com/v1${fetchPath}`;  
  const fetchHeaders = getHeadersAsObject(request.headers);  
  const fetchBody = await request.text();  
  const forceRefresh = request.headers.get('X-Proxy-Refresh') === 'true';  

  const cacheKey = await getCacheKey({  
    authHeader: request.headers.get('authorization'),  
    body: fetchBody,  
    method: fetchMethod,  
    path: fetchPath,  
  });  
  const responseCache = new ResponseCache({ env });  

  if (forceRefresh) {  
    console.log('X-Proxy-Refresh was true, forcing a cache refresh.');  
  } else {  
    const cachedResponse = await responseCache.read({ cacheKey });  
    if (cachedResponse) {  
      console.log('Returning cached response.');  
      return cachedResponse;  
    }  
  }  

  const response = await fetch(fetchUrl, {  
    method: fetchMethod,  
    headers: fetchHeaders,  
    body: fetchBody || null,  
  });  

  if (response.ok) {  
    console.log('Writing 2xx response to cache: ', { cacheKey, ttl });  
    const writeCachePromise = responseCache.write({  
      cacheKey,  
      ttl,  
      response,  
    });  
    ctx.waitUntil(writeCachePromise);  
  } else {  
    console.log('Not caching error or empty response.');  
  }  

  return response;  
};  
```  

This is my current file. The insertion point will be denoted by the comments: Start Generation Here and End Generation Here.  
```typescript  
import { Env } from './env';  
import { handleProxy } from './proxy';  

export default {  
  async fetch(request: Request, env: Env, ctx: ExecutionContext): Promise<Response> {  
    const { pathname } = new URL(request.url);  
    console.log(`Received request for pathname: ${pathname}`);  

    if (request.method === 'POST') {  
      const requestBody = await request.clone().text();  
      console.log({ requestBody });  
      try {  
        const parsedRequestBody = JSON.parse(requestBody);  
        console.log({ parsedRequestBody });  
      } catch (error) {  
        console.error('Failed to parse request body as JSON:', error);  
      }  
    }  
    const response = await handleProxy({  
      request,  
      env,  
      ctx,  
      ttl: null,  
      pathname,  
    });  
    const responseBody = await response.clone().text();  
    console.log('Response Body:', responseBody);  
    return response;  
  },  
};  

// Start Generation Here  
// End Generation Here  
```"