const express = require('express');
const fetch = require('node-fetch');
const cors = require('cors');
require('dotenv').config();

const app = express();
const PORT = process.env.PORT || 3000;

app.use(cors());

const currencies = ['USD', 'EUR', 'GBP', 'JPY', 'AUD'];

const sentimentMap = {
  positive: ['growth', 'expansion', 'rate hike', 'optimism', 'bullish', 'strong'],
  negative: ['recession', 'unemployment', 'rate cut', 'inflation', 'bearish', 'weak']
};

const apiKey = process.env.MARKETAUX_API_KEY;

function buildNewsUrl() {
  return `https://api.marketaux.com/v1/news/all?filter_entities=true&language=en&limit=50&api_token=${apiKey}`;
}

function scoreHeadline(headline, currency) {
  const upper = headline.toUpperCase();
  if (!upper.includes(currency)) return 0;

  let score = 0;
  sentimentMap.positive.forEach(word => {
    if (upper.includes(word.toUpperCase())) score += 1;
  });
  sentimentMap.negative.forEach(word => {
    if (upper.includes(word.toUpperCase())) score -= 1;
  });
  return score;
}

async function getCurrencyStrengths() {
  const url = buildNewsUrl();
  const res = await fetch(url);
  if (!res.ok) {
    throw new Error(`Failed to fetch news data: ${res.status}`);
  }
  const json = await res.json();
  const articles = json.data || [];

  const strength = {};
  currencies.forEach(cur => {
    let total = 0;
    articles.forEach(article => {
      total += scoreHeadline(article.title || '', cur);
    });
    strength[cur] = total;
  });
  return strength;
}

app.get('/strength', async (req, res) => {
  try {
    const data = await getCurrencyStrengths();
    res.json(data);
  } catch (err) {
    console.error('Error fetching or processing news:', err);
    res.status(500).json({ error: 'Failed to get strength data' });
  }
});

app.listen(PORT, () => console.log(`🚀 News API running at http://localhost:${PORT}`));
