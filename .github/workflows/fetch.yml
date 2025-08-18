import { TwitterApi } from 'twitter-api-v2';
import fs from 'fs';
import dayjs from 'dayjs';

const BEARER = process.env.X_BEARER_TOKEN;
if (!BEARER) {
  console.error('Missing X_BEARER_TOKEN env var');
  process.exit(1);
}
const client = new TwitterApi(BEARER);

// --- Edit/extend this alias map anytime ---
const COINS = [
  { symbol: "HYPE", aliases: ['Hyperliquid', '$HYPE', '#HYPE'] },
  { symbol: "LINK", aliases: ['Chainlink', '$LINK', '#LINK'] },
  { symbol: "JUP",  aliases: ['Jupiter', '$JUP', '#JUP', 'JLP'] },
  { symbol: "KMNO", aliases: ['Kamino', '$KMNO', '#KMNO'] },
  { symbol: "TOWNS",aliases: ['Towns', '$TOWNS', '#TOWNS'] },
  { symbol: "IP",   aliases: ['Story Protocol', 'Story', '$IP', '#IP'] },
  { symbol: "BTC",  aliases: ['Bitcoin', '$BTC', '#BTC'] },
  { symbol: "ETH",  aliases: ['Ethereum', '$ETH', '#ETH'] }
];

const WINDOW_DAYS = 7;                     // last 7d window
const sinceISO = dayjs().subtract(WINDOW_DAYS, 'day').toISOString();

// Helper: build X query with our anti-spam filters
const term = t => (t.startsWith('$') || t.startsWith('#')) ? t : `"${t}"`;
const buildQuery = (aliases) =>
  `(${aliases.map(term).join(' OR ')}) lang:en -is:retweet -is:reply ` +
  `-has:cashtags ("presale" OR "airdrop" OR "giveaway" OR "bonding curve" OR "stealth launch") (min_faves:5 OR has:mentions OR has:links)`;

const TWEET_FIELDS = ['created_at','lang','public_metrics','possibly_sensitive'];
const USER_FIELDS  = ['username','name','verified','public_metrics'];
const EXPANSIONS   = ['author_id'];

async function fetchForCoin(coin) {
  const q = buildQuery(coin.aliases);
  const paginator = await client.v2.search(q, {
    'start_time': sinceISO,
    'max_results': 100,
    'tweet.fields': TWEET_FIELDS.join(','),
    'user.fields': USER_FIELDS.join(','),
    'expansions': EXPANSIONS.join(',')
  });

  const users = new Map();
  const rows = [];
  const pushUsers = (arr=[]) => arr.forEach(u => users.set(u.id, u));

  // Collect up to ~300 tweets/coin to stay within rate limits
  let pulled = 0;
  pushUsers(paginator.includes?.users);
  for await (const tweet of paginator) {
    const u = users.get(tweet.author_id);
    const pm = tweet.public_metrics || {};
    rows.push({
      id: tweet.id,
      created_at: tweet.created_at,
      text: tweet.text,
      like_count: pm.like_count || 0,
      retweet_count: pm.retweet_count || 0,
      reply_count: pm.reply_count || 0,
      quote_count: pm.quote_count || 0,
      author_id: tweet.author_id,
      author_username: u?.username,
      author_verified: !!u?.verified,
      author_followers: u?.public_metrics?.followers_count || 0
    });
    if (++pulled >= 300) break;
  }

  // Aggregate
  const uniqVerified = new Set(rows.filter(r => r.author_verified).map(r => r.author_id));
  const over100k = new Set(rows.filter(r => r.author_followers >= 100000).map(r => r.author_id));
  const engagements = rows.reduce((s, r) => s + r.like_count + r.retweet_count + r.reply_count + r.quote_count, 0);

  // Top tweets by engagement (for downstream display)
  const topTweets = rows
    .sort((a,b) => (b.like_count+b.retweet_count+b.reply_count+b.quote_count) -
                   (a.like_count+a.retweet_count+a.reply_count+a.quote_count))
    .slice(0, 15)
    .map(r => ({
      id: r.id,
      url: r.author_username ? `https://twitter.com/${r.author_username}/status/${r.id}` : `https://twitter.com/i/web/status/${r.id}`,
      author_username: r.author_username,
      verified: r.author_verified,
      followers: r.author_followers,
      engagement: r.like_count + r.retweet_count + r.reply_count + r.quote_count,
      created_at: r.created_at
    }));

  return {
    symbol: coin.symbol,
    query: q,
    window_start: sinceISO,
    mentions: rows.length,
    unique_verified_accounts: uniqVerified.size,
    unique_100kplus_accounts: over100k.size,
    total_engagements: engagements,
    top_tweets: topTweets
  };
}

(async () => {
  const out = { generated_at: new Date().toISOString(), window_days: WINDOW_DAYS, coins: [] };
  for (const c of COINS) {
    try {
      const data = await fetchForCoin(c);
      out.coins.push(data);
    } catch (e) {
      out.coins.push({ symbol: c.symbol, error: String(e) });
    }
  }
  fs.writeFileSync('feed.json', JSON.stringify(out, null, 2), 'utf8');
  console.log('Wrote feed.json with', out.coins.length, 'coins');
})();
