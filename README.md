# Crypto-Tracker
from flask import Flask, request, render_template_string
import requests
from waitress import serve
import json

app = Flask(__name__)

TEMPLATE = """
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>🌐 Crypto Market Tracker</title>
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    <style>
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            background: linear-gradient(to right, #e0f7fa, #e1bee7);
            margin: 0;
            padding: 20px;
            color: #333;
        }
        h1 {
            text-align: center;
            color: #4a148c;
            font-size: 2.5em;
            margin-bottom: 10px;
        }
        form {
            text-align: center;
            margin-bottom: 30px;
        }
        input[type="text"] {
            padding: 10px;
            width: 250px;
            border: 2px solid #7b1fa2;
            border-radius: 10px;
            font-size: 16px;
        }
        button {
            padding: 10px 20px;
            background-color: #7b1fa2;
            color: white;
            border: none;
            border-radius: 10px;
            font-size: 16px;
            cursor: pointer;
        }
        button:hover {
            background-color: #4a148c;
        }
        table {
            width: 90%;
            margin: 0 auto;
            border-collapse: collapse;
            background-color: #ffffff;
            border-radius: 10px;
            overflow: hidden;
            box-shadow: 0 4px 10px rgba(0,0,0,0.1);
        }
        th {
            background-color: #ba68c8;
            color: white;
            padding: 15px;
        }
        td {
            padding: 15px;
            text-align: center;
            border-bottom: 1px solid #eee;
        }
        .rising {
            color: green;
            font-weight: bold;
        }
        .falling {
            color: red;
            font-weight: bold;
        }
        h2 {
            color: #6a1b9a;
            margin-top: 50px;
        }
        h3 {
            color: #8e24aa;
        }
        canvas {
            max-width: 600px;
            margin: 20px auto;
            background-color: #ffffff;
            padding: 10px;
            border-radius: 12px;
            box-shadow: 0 2px 8px rgba(0,0,0,0.1);
        }
        .error {
            text-align: center;
            color: red;
            font-weight: bold;
            margin-top: 20px;
        }
    </style>
</head>
<body>
    <h1>📈 Crypto Market Trends</h1>
    <form method="get">
        <input type="text" name="search" placeholder="Enter cryptocurrency name..." value="{{ search_query }}">
        <button type="submit">Search</button>
    </form>
    
    {% if trends %}
        <table>
            <tr>
                <th>Coin</th>
                <th>Previous Price (USD)</th>
                <th>Current Price (USD)</th>
                <th>Trend</th>
            </tr>
            {% for coin, data in trends.items() %}
            <tr>
                <td>{{ coin.capitalize() }}</td>
                <td>${{ data.previous_price }}</td>
                <td>${{ data.current_price }}</td>
                <td class="{{ 'rising' if data.trend == '🔼 Rising' else 'falling' }}">{{ data.trend }}</td>
            </tr>
            {% endfor %}
        </table>

        <h2>📊 Price Graphs (Last 7 Days)</h2>
        {% for coin, data in trends.items() %}
        <h3>{{ coin.capitalize() }}</h3>
        <canvas id="chart{{ loop.index }}"></canvas>
        <script>
            const ctx{{ loop.index }} = document.getElementById('chart{{ loop.index }}').getContext('2d');
            new Chart(ctx{{ loop.index }}, {
                type: 'line',
                data: {
                    labels: Array.from({length: {{ data.sparkline|length }} }, (_, i) => i),
                    datasets: [{
                        label: '{{ coin.capitalize() }} Price (USD)',
                        data: {{ data.sparkline }},
                        borderColor: '{{ "green" if data.trend == "🔼 Rising" else "red" }}',
                        backgroundColor: 'rgba(255,255,255,0.5)',
                        fill: false,
                        tension: 0.3
                    }]
                },
                options: {
                    responsive: true,
                    plugins: {
                        legend: { display: false }
                    },
                    scales: {
                        x: { display: false },
                        y: { beginAtZero: false }
                    }
                }
            });
        </script>
        {% endfor %}
    {% else %}
        <p class="error">⚠️ Failed to fetch data. Please try again later.</p>
    {% endif %}
</body>
</html>
"""

@app.route('/')
def home():
    search_query = request.args.get('search', '').lower()
    trends = {}

    try:
        api_url = "https://api.coingecko.com/api/v3/coins/markets"
        params = {
            "vs_currency": "usd",
            "order": "market_cap_desc",
            "per_page": 50,
            "page": 1,
            "sparkline": "true"
        }
        response = requests.get(api_url, params=params, timeout=10)
        response.raise_for_status()

        for coin in response.json():
            coin_id = coin.get("id", "")
            if search_query and search_query not in coin_id:
                continue

            prices = coin.get("sparkline_in_7d", {}).get("price", [])
            if not prices or len(prices) < 2:
                continue

            previous_price = round(prices[-2], 2)
            current_price = round(coin.get("current_price", 0), 2)
            trend = "🔼 Rising" if current_price > previous_price else "🔽 Falling"

            trends[coin_id] = {
                "previous_price": previous_price,
                "current_price": current_price,
                "trend": trend,
                "sparkline": json.dumps([round(p, 2) for p in prices[-50:]])  # Reduce to last 50 points
            }

        if not search_query:
            trends = dict(list(trends.items())[:5])  # Show top 5 by default

    except Exception as e:
        print("API Fetch Error:", e)
        trends = {}

    return render_template_string(TEMPLATE, trends=trends, search_query=search_query)

if __name__ == '__main__':
    print("Server running on http://127.0.0.1:8000/")
    serve(app, host='0.0.0.0', port=8000)
