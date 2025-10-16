# MiniPay-AllInOne
MiniPay-AllInOne Descripción: MiniPay-AllInOne es una plataforma de pagos todo-en-uno desarrollada en un solo archivo Node.js, diseñada para administradores que desean recibir pagos directamente de clientes sin depender de terceros como PayPal.
// mini-pay-modern.js
require('dotenv').config();
const express = require('express');
const bodyParser = require('body-parser');
const bcrypt = require('bcrypt');
const jwt = require('jsonwebtoken');
const mercadopago = require('mercadopago');
const { Pool } = require('pg');
const cors = require('cors');

const app = express();
app.use(cors());
app.use(bodyParser.json());

// --- Config DB ---
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// --- Config Mercado Pago ---
mercadopago.configurations.setAccessToken(process.env.MP_ACCESS_TOKEN);

// --- Middleware auth ---
const auth = async (req,res,next)=>{
    const h = req.headers.authorization;
    if(!h) return res.status(401).send({error:'No auth'});
    const token = h.split(' ')[1];
    try{
        const payload = jwt.verify(token,process.env.JWT_SECRET);
        req.user = payload;
        next();
    }catch(e){
        res.status(401).send({error:'Invalid token'});
    }
};

// --- Inicializar DB ---
async function initDB(){
    await pool.query(`
    CREATE TABLE IF NOT EXISTS users(
        id SERIAL PRIMARY KEY,
        email VARCHAR(255) UNIQUE NOT NULL,
        password_hash VARCHAR(255) NOT NULL,
        role VARCHAR(50) DEFAULT 'admin',
        created_at TIMESTAMP DEFAULT NOW()
    );
    CREATE TABLE IF NOT EXISTS transactions(
        id SERIAL PRIMARY KEY,
        user_id INT REFERENCES users(id),
        type VARCHAR(50) NOT NULL,
        amount_cents INT NOT NULL,
        currency VARCHAR(10) DEFAULT 'USD',
        status VARCHAR(50) DEFAULT 'pending',
        provider_id VARCHAR(255),
        note TEXT,
        created_at TIMESTAMP DEFAULT NOW()
    );
    CREATE TABLE IF NOT EXISTS ledger(
        id SERIAL PRIMARY KEY,
        user_id INT REFERENCES users(id),
        balance_cents INT NOT NULL DEFAULT 0,
        updated_at TIMESTAMP DEFAULT NOW()
    );
    `);
    console.log('Tablas inicializadas.');
}

// --- Crear admin inicial ---
async function initAdmin(){
    const {ADMIN_EMAIL, ADMIN_PASSWORD} = process.env;
    const r = await pool.query('SELECT * FROM users WHERE email=$1',[ADMIN_EMAIL]);
    if(r.rows.length===0){
        const hash = await bcrypt.hash(ADMIN_PASSWORD,10);
        await pool.query('INSERT INTO users(email,password_hash,role) VALUES($1,$2,$3)',
            [ADMIN_EMAIL,hash,'admin']);
        console.log(`Admin creado: ${ADMIN_EMAIL}`);
    }
}

// --- Login admin ---
app.post('/api/login', async (req,res)=>{
    const {email,password} = req.body;
    const r = await pool.query('SELECT * FROM users WHERE email=$1',[email]);
    if(r.rows.length===0) return res.status(401).send({error:'Usuario no encontrado'});
    const user = r.rows[0];
    const valid = await bcrypt.compare(password,user.password_hash);
    if(!valid) return res.status(401).send({error:'Contraseña incorrecta'});
    const token = jwt.sign({id:user.id,email:user.email},process.env.JWT_SECRET,{expiresIn:'12h'});
    res.send({token});
});

// --- Crear pago Mercado Pago ---
app.post('/api/create-payment', auth, async (req,res)=>{
    const {amount, description} = req.body;
    try{
        const preference = {
            items: [{ title: description, unit_price: parseFloat(amount), quantity: 1 }],
            back_urls: { success: "/", failure: "/", pending: "/" },
            auto_return: "approved"
        };
        const response = await mercadopago.preferences.create(preference);
        await pool.query(
            'INSERT INTO transactions(user_id,type,amount_cents,currency,status,provider_id,note) VALUES($1,$2,$3,$4,$5,$6,$7)',
            [req.user.id,'card',Math.round(amount*100),'USD','pending',response.body.id,description]
        );
        res.send({init_point: response.body.init_point});
    }catch(err){res.status(500).send({error:err.message});}
});

// --- Depósito bancario manual ---
app.post('/api/deposit', auth, async (req,res)=>{
    const {amount,note} = req.body;
    await pool.query(
        'INSERT INTO transactions(user_id,type,amount_cents,currency,status,note) VALUES($1,$2,$3,$4,$5,$6)',
        [req.user.id,'bank',Math.round(amount*100),'USD','confirmed',note]
    );
    const r = await pool.query('SELECT * FROM ledger WHERE user_id=$1 ORDER BY id DESC LIMIT 1',[req.user.id]);
    const balance = r.rows.length>0?r.rows[0].balance_cents:0;
    const newBalance = balance + Math.round(amount*100);
    await pool.query('INSERT INTO ledger(user_id,balance_cents) VALUES($1,$2)',[req.user.id,newBalance]);
    res.send({success:true,newBalance});
});

// --- Listar transacciones y saldo ---
app.get('/api/transactions', auth, async (req,res)=>{
    const txs = await pool.query('SELECT * FROM transactions WHERE user_id=$1 ORDER BY created_at DESC',[req.user.id]);
    const l = await pool.query('SELECT * FROM ledger WHERE user_id=$1 ORDER BY id DESC LIMIT 1',[req.user.id]);
    res.send({transactions:txs.rows, balance:l.rows.length>0?l.rows[0].balance_cents:0});
});

// --- Solicitar retiro ---
app.post('/api/withdraw', auth, async (req,res)=>{
    const {amount, method, details} = req.body;
    const l = await pool.query('SELECT * FROM ledger WHERE user_id=$1 ORDER BY id DESC LIMIT 1',[req.user.id]);
    const balance = l.rows.length>0?l.rows[0].balance_cents:0;
    if(balance<Math.round(amount*100)) return res.status(400).send({error:'Saldo insuficiente'});
    await pool.query(
        'INSERT INTO transactions(user_id,type,amount_cents,currency,status,note) VALUES($1,$2,$3,$4,$5,$6)',
        [req.user.id,'withdrawal',Math.round(amount*100),'USD','pending',`Retiro: ${method}, ${details}`]
    );
    const newBalance = balance - Math.round(amount*100);
    await pool.query('INSERT INTO ledger(user_id,balance_cents) VALUES($1,$2)',[req.user.id,newBalance]);
    res.send({success:true,newBalance});
});

// --- Interfaz web moderna ---
app.get('/', (req,res)=>{
    res.send(`
<!DOCTYPE html>
<html lang="es">
<head>
<meta charset="UTF-8">
<title>MiniPay Dashboard</title>
<style>
body { font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; background:#f5f7fa; margin:0; padding:0;}
header { background:#4a90e2; color:white; padding:1rem; text-align:center;}
.container { max-width:900px; margin:20px auto; padding:20px; background:white; border-radius:10px; box-shadow:0 2px 10px rgba(0,0,0,0.1);}
input, button { padding:10px; margin:5px 0; border-radius:5px; border:1px solid #ccc;}
button { cursor:pointer; background:#4a90e2; color:white; border:none;}
button:hover { background:#357ab8;}
.card { background:#e9f0ff; padding:15px; border-radius:8px; margin-bottom:15px;}
.tx-list { list-style:none; padding:0;}
.tx-list li { padding:8px; margin-bottom:5px; border-left:5px solid #ccc; background:#f9f9f9; border-radius:5px;}
.tx-pending { border-left-color:#f5a623;}
.tx-approved { border-left-color:#7ed321;}
.tx-failed { border-left-color:#d0021b;}
@media(max-width:600px){.container{margin:10px; padding:10px;}}
</style>
<script src="https://sdk.mercadopago.com/js/v2"></script>
</head>
<body>
<header><h1>MiniPay Dashboard</h1></header>
<div class="container">
<h2>Admin Login</h2>
<input id="email" placeholder="Email"/><br/>
<input id="pass" type="password" placeholder="Contraseña"/><br/>
<button onclick="login()">Login</button>
<hr/>
<div id="panel" style="display:none">
<div class="card"><h3>Saldo: <span id="balance">0</span> USD</h3></div>
<button onclick="loadTransactions()">Actualizar transacciones</button>
<ul id="txs" class="tx-list"></ul>
<h3>Crear pago</h3>
<input id="payAmount" placeholder="Monto"/>
<input id="payDesc" placeholder="Descripción"/>
<button onclick="createPayment()">Cobrar</button>
<h3>Depósito manual</h3>
<input id="depAmount" placeholder="Monto"/>
<input id="depNote" placeholder="Nota"/>
<button onclick="deposit()">Agregar</button>
<h3>Retiro</h3>
<input id="withAmount" placeholder="Monto"/>
<input id="withMethod" placeholder="Banco/PayPal"/>
<input id="withDetails" placeholder="Detalles"/>
<button onclick="withdraw()">Retirar</button>
</div>
</div>
<script>
let token='';
const mp = new MercadoPago('TEST-ACCESS-TOKEN',{locale:'es-EC'});
function login(){
    fetch('/api/login',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({email:document.getElementById('email').value,password:document.getElementById('pass').value})})
    .then(r=>r.json()).then(d=>{if(d.token){token=d.token; document.getElementById('panel').style.display='block'; alert('Login correcto');}else{alert('Error login')}})
}
function loadTransactions(){
    fetch('/api/transactions',{headers:{Authorization:'Bearer '+token}})
    .then(r=>r.json()).then(d=>{
        document.getElementById('balance').innerText=(d.balance/100).toFixed(2);
        const txs = d.transactions.map(t=>{
            let cls = t.status==='pending'?'tx-pending': t.status==='approved'?'tx-approved':'tx-failed';
            return '<li class="'+cls+'">'+t.type+': '+(t.amount_cents/100)+' USD - '+t.status+'</li>';
        }).join('');
        document.getElementById('txs').innerHTML=txs;
    })
}
function createPayment(){
    const a=parseFloat(document.getElementById('payAmount').value);
    const desc=document.getElementById('payDesc').value;
    fetch('/api/create-payment',{method:'POST',headers:{'Content-Type':'application/json','Authorization':'Bearer '+token},body:JSON.stringify({amount:a,description:desc})})
    .then(r=>r.json()).then(d=>{
        window.open(d.init_point,'_blank');
        loadTransactions();
    })
}
function deposit(){
    const a=parseFloat(document.getElementById('depAmount').value);
    const note=document.getElementById('depNote').value;
    fetch('/api/deposit',{method:'POST',headers:{'Content-Type':'application/json','Authorization':'Bearer '+token},body:JSON.stringify({amount:a,note})})
    .then(r=>r.json()).then(d=>{alert('Depósito agregado'); loadTransactions();})
}
function withdraw(){
    const a=parseFloat(document.getElementById('withAmount').value);
    const m=document.getElementById('withMethod').value;
    const d=document.getElementById('withDetails').value;
    fetch('/api/withdraw',{method:'POST',headers:{'Content-Type':'application/json','Authorization':'Bearer '+token},body:JSON.stringify({amount:a,method:m,details:d})})
    .then(r=>r.json()).then(d=>{alert('Retiro solicitado'); loadTransactions();})
}
</script>
</body>
</html>
    `);
});

// --- Inicializar DB y admin ---
initDB().then(initAdmin);

app.listen(3000,()=>console.log('MiniPay moderno corriendo en http://localhost:3000'));
