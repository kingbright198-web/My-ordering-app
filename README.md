# My-ordering-app
The promise 
const express = require('express');
const fs = require('fs');
const app = express();
const PORT = 3000;

app.use(express.json());

// --- DATABASE LOGIC ---
const DATA_FILE = './orders.json';
if (!fs.existsSync(DATA_FILE)) fs.writeFileSync(DATA_FILE, '[]');

const getOrders = () => JSON.parse(fs.readFileSync(DATA_FILE));
const saveOrders = (orders) => fs.writeFileSync(DATA_FILE, JSON.stringify(orders, null, 2));

// --- API ROUTES ---
app.post('/api/order', (req, res) => {
    const orders = getOrders();
    const newOrder = { id: Date.now(), ...req.body, time: new Date().toLocaleTimeString() };
    orders.push(newOrder);
    saveOrders(orders);
    res.json({ success: true });
});

app.get('/api/orders', (res) => res.json(getOrders()));

app.delete('/api/order/:id', (req, res) => {
    let orders = getOrders().filter(o => o.id != req.params.id);
    saveOrders(orders);
    res.sendStatus(200);
});

// --- FRONTEND: CUSTOMER MENU ---
app.get('/', (req, res) => {
    res.send(`
        <!DOCTYPE html>
        <html>
        <head>
            <title>On-Premise Ordering</title>
            <meta name="viewport" content="width=device-width, initial-scale=1">
            <style>
                body { font-family: sans-serif; margin: 0; padding: 20px; background: #f8f9fa; }
                .item { background: white; padding: 15px; margin-bottom: 10px; border-radius: 8px; display: flex; justify-content: space-between; align-items: center; box-shadow: 0 2px 4px rgba(0,0,0,0.1); }
                #cart { position: sticky; bottom: 0; background: #333; color: white; padding: 20px; border-radius: 15px 15px 0 0; margin: -20px; margin-top: 20px; }
                button { background: #28a745; color: white; border: none; padding: 10px 20px; border-radius: 5px; cursor: pointer; }
                textarea { width: 100%; margin: 10px 0; border-radius: 5px; border: none; padding: 8px; }
            </style>
        </head>
        <body>
            <h1>Menu</h1>
            <input type="text" id="tbl" placeholder="Table #" style="width: 100%; padding: 10px; margin-bottom: 20px;">
            
            <div class="item">
                <div><strong>Espresso</strong><br>$3.00</div>
                <button onclick="add('Espresso', 3.00)">Add</button>
            </div>
            <div class="item">
                <div><strong>Club Sandwich</strong><br>$8.50</div>
                <button onclick="add('Club Sandwich', 8.50)">Add</button>
            </div>

            <div id="cart">
                <h3>Your Cart</h3>
                <div id="items"></div>
                <textarea id="note" placeholder="Special instructions..."></textarea>
                <p>Total: $<span id="total">0.00</span></p>
                <button style="width:100%; background:#007bff" onclick="send()">Place Order</button>
            </div>

            <script>
                let cart = [];
                function add(n, p) { cart.push({name:n, price:p}); render(); }
                function render() {
                    document.getElementById('items').innerHTML = cart.map(i => i.name).join(', ');
                    document.getElementById('total').innerText = cart.reduce((s, i) => s + i.price, 0).toFixed(2);
                }
                async function send() {
                    const table = document.getElementById('tbl').value;
                    if(!table || !cart.length) return alert("Missing info");
                    await fetch('/api/order', {
                        method: 'POST',
                        headers: {'Content-Type': 'application/json'},
                        body: JSON.stringify({ location: table, items: cart, notes: document.getElementById('note').value, total: cart.reduce((s,i)=>s+i.price,0) })
                    });
                    alert("Order Sent!");
                    cart = []; render();
                }
            </script>
        </body>
        </html>
    `);
});

// --- FRONTEND: ADMIN DASHBOARD ---
app.get('/admin', (req, res) => {
    if (req.query.password !== "123") return res.send("Access Denied");
    res.send(`
        <!DOCTYPE html>
        <html>
        <head>
            <title>Kitchen</title>
            <style>
                body { font-family: sans-serif; background: #222; color: white; padding: 20px; }
                .card { background: #444; padding: 15px; margin: 10px 0; border-left: 10px solid #28a745; display: flex; justify-content: space-between; }
                .notes { color: #ffc107; font-style: italic; }
            </style>
        </head>
        <body>
            <h1>Kitchen Dashboard</h1>
            <div id="list"></div>
            <script>
                async function load() {
                    const res = await fetch('/api/orders');
                    const orders = await res.json();
                    document.getElementById('list').innerHTML = orders.map(o => \`
                        <div class="card">
                            <div>
                                <strong>\${o.location}</strong> - \${o.time}<br>
                                <span>\${o.items.map(i => i.name).join(', ')}</span><br>
                                <span class="notes">\${o.notes || ''}</span>
                            </div>
                            <button onclick="done('\${o.id}')">Complete</button>
                        </div>
                    \`).join('');
                }
                async function done(id) {
                    await fetch('/api/order/' + id, { method: 'DELETE' });
                    load();
                }
                setInterval(load, 5000);
                load();
            </script>
        </body>
        </html>
    `);
});

app.listen(PORT, () => console.log(\`Server Live! Customer: http://localhost:\${PORT} | Admin: http://localhost:\${PORT}/admin?password=123\`));
