
require('dotenv').config();
const express = require('express');
const axios = require('axios');
const OpenAI = require('openai');

const app = express();
app.use(express.json());

const VERIFY_TOKEN = process.env.VERIFY_TOKEN || 'geskap_verify_123';
const WHATSAPP_TOKEN = process.env.WHATSAPP_TOKEN;
const PHONE_NUMBER_ID = process.env.PHONE_NUMBER_ID;
const MTN_MOMO_NUMBER = process.env.MTN_MOMO_NUMBER || '06 927 30 22'; // Ton numero marchand
const ORANGE_MONEY_NUMBER = process.env.ORANGE_MONEY_NUMBER || '06 927 30 22';
const GOOGLE_SHEET_WEBHOOK = process.env.GOOGLE_SHEET_WEBHOOK_URL;

const openai = process.env.OPENAI_API_KEY ? new OpenAI({ apiKey: process.env.OPENAI_API_KEY }) : null;
const sessions = {};

// === WEBHOOK VERIFICATION ===
app.get('/webhook', (req, res) => {
  if (req.query['hub.mode'] === 'subscribe' && req.query['hub.verify_token'] === VERIFY_TOKEN) {
    return res.status(200).send(req.query['hub.challenge']);
  }
  res.sendStatus(403);
});

app.post('/webhook', async (req, res) => {
  const message = req.body.entry?.[0]?.changes?.[0]?.value?.messages?.[0];
  if (!message) return res.sendStatus(200);
  
  const from = message.from;
  const text = (message.text?.body || '').toLowerCase();
  const buttonId = message.interactive?.button_reply?.id || '';
  
  if (!sessions[from]) sessions[from] = { step: 'start', data: { phone: from } };

  try {
    if (sessions[from].step === 'start' || ['bonjour','salut','hello','prix','info','cc'].some(k => text.includes(k))) {
      await sendText(from, "Je vois ! C'est souvent le cas quand on commence à vendre en ligne. On veut bien gérer, mais les messages arrivent de partout et on finit par en laisser passer.");
      await delay(700);
      await sendText(from, "Chaque message sans réponse, c'est une vente qui part chez un concurrent. 😕");
      await delay(700);
      await sendButtons(from, "Moi c'est Geskap, ton vendeur auto sur WhatsApp 😊\n\nTu travailles seul ou tu as une boutique / équipe ?", [
        { id: 'seul', title: '🧍 Seul' },
        { id: 'equipe', title: '🏪 Boutique' }
      ]);
      sessions[from].step = 'qualif';
      return res.sendStatus(200);
    }

    if (sessions[from].step === 'qualif') {
      let choice = buttonId || (text.includes('seul') ? 'seul' : (text.includes('boutique')||text.includes('equipe')?'equipe':''));
      if (choice === 'seul') {
        sessions[from].data.type = 'SOLO';
        sessions[from].data.amount = 10000;
        await sendText(from, "Parfait solo ! Bot qui répond en 30s même à 2h du matin, catalogue auto, encaissement MTN MoMo / Orange Money auto.\n\nOption 1 - SOLO:\n✅ 10 000 FCFA config 1-on-1\n✅ 30 jours d'essai gratuits\n✅ Installé en 1h");
        await sendButtons(from, "On valide ta place ?", [
          { id: 'payer_seul', title: '⚡ Payer 10 000 F' },
          { id: 'question', title: '❓ Question' }
        ]);
        sessions[from].step = 'close';
      } else if (choice === 'equipe') {
        sessions[from].data.type = 'BOUTIQUE';
        sessions[from].data.amount = 25000;
        await sendText(from, "Top boutique ! Distribution auto des chats, suivi vendeurs, stock synchro.\n\nOption 2 - BOUTIQUE:\n✅ 25 000 FCFA multi-vendeurs\n✅ Tableau de bord\n✅ 30 jours gratuits");
        await sendButtons(from, "On lance ?", [
          { id: 'payer_equipe', title: '⚡ Payer 25 000 F' },
          { id: 'question', title: '❓ Question' }
        ]);
        sessions[from].step = 'close';
      }
      return res.sendStatus(200);
    }

    // === PAIEMENT CONGO - MODE MANUEL + AUTO ===
    if (sessions[from].step === 'close' && buttonId.startsWith('payer')) {
      const amount = sessions[from].data.amount;
      const type = sessions[from].data.type;
      
      // Mode 1: Si MTN API configurée, on lance un Request to Pay automatique
      if (process.env.MTN_MOMO_API_KEY) {
        const momo = await requestMTNMoMoPay(from, amount);
        if (momo.success) {
          await sendText(from, `J'ai lancé une demande de paiement de ${amount} FCFA sur ton MTN MoMo.\n\n👉 Valide avec ton code PIN MoMo sur ton téléphone. Dès que c'est ok je confirme ici.`);
          await logToSheet({ phone: from, type, amount, status: 'MTN_REQUEST_SENT', date: new Date().toLocaleString('fr-FR') });
          sessions[from].step = 'attente_momo';
          return res.sendStatus(200);
        }
      }

      // Mode 2: Manuel Congo - marche immediatement sans API
      await sendText(from, `Parfait ! Pour valider ta config Geskap *${type}* - *${amount} FCFA*, paie comme tu veux :\n\n` +
        `📱 *MTN MoMo* :\n` +
        `*105*1*1*${MTN_MOMO_NUMBER.replace(/\s/g,'')}*${amount}# puis code PIN\n` +
        `Ou Paiement Marchand : *105# -> 2 Facture -> 2 Marchand -> Code: ${MTN_MOMO_NUMBER}\n\n` +
        `🟠 *Orange Money* :\n` +
        `*144*1*1*${ORANGE_MONEY_NUMBER.replace(/\s/g,'')}*${amount}#\n\n` +
        `Après paiement, envoie ici :\n` +
        `1. La capture d'écran\n` +
        `2. Ou l'ID transaction (ex: ID: 12345678)\n\n` +
        `Je vérifie et je lance ta config en 15 min ⚡`);

      await sendButtons(from, "Tu as payé ?", [
        { id: 'confirme_paiement', title: '✅ J\'ai payé' },
        { id: 'aide_paiement', title: '❓ Aide paiement' }
      ]);

      await logToSheet({ phone: from, type, amount, status: 'LIEN_MANUEL_ENVOYE', date: new Date().toLocaleString('fr-FR') });
      sessions[from].step = 'attente_preuve';
      return res.sendStatus(200);
    }

    if (sessions[from].step === 'attente_preuve') {
      if (buttonId === 'confirme_paiement' || text.includes('payé') || text.includes('payer') || text.includes('id:') || text.length > 6) {
        await sendText(from, "✅ Reçu ! Merci. Notre agent vérifie ton paiement MTN/Orange en moins de 15 min et lance ton installation Geskap.\n\nEn attendant, envoie ton nom de boutique et ce que tu vends.");
        await logToSheet({ phone: from, type: sessions[from].data.type, amount: sessions[from].data.amount, status: 'PREUVE_RECU: ' + text.slice(0,100), date: new Date().toLocaleString('fr-FR') });
        // Ici tu peux notifier ton propre WhatsApp
        await notifyOwner(`💰 Nouveau paiement Geskap ! ${from} - ${sessions[from].data.type} - ${sessions[from].data.amount}F - Preuve: ${text}`);
        sessions[from].step = 'termine';
      } else if (buttonId === 'aide_paiement') {
        await sendText(from, "Pas de souci : va dans MTN MoMo -> *105# -> Envoi Argent -> Mets mon numéro " + MTN_MOMO_NUMBER + " -> Montant " + sessions[from].data.amount + ". Même chose Orange *144#. Envoie la capture après.");
      }
      return res.sendStatus(200);
    }

    // IA fallback
    const aiReply = await askAI(text, sessions[from].data);
    await sendText(from, aiReply);

  } catch (e) {
    console.error(e.response?.data || e.message);
  }
  res.sendStatus(200);
});

// === MTN MOMO DIRECT API CONGO ===
async function requestMTNMoMoPay(phone, amount) {
  // Documentation: https://momodeveloper.mtn.com - Collection API
  // Besoin: SUBSCRIPTION_KEY, API_USER, API_KEY
  try {
    // 1. Get token
    // 2. Request to pay
    // Simplifié ici - à compléter avec tes clés MTN Congo
    return { success: false };
  } catch { return { success: false }; }
}

async function sendText(to, text) {
  return axios.post(`https://graph.facebook.com/v20.0/${PHONE_NUMBER_ID}/messages`, {
    messaging_product: 'whatsapp',
    to,
    text: { body: text }
  }, { headers: { Authorization: `Bearer ${WHATSAPP_TOKEN}` } });
}

async function sendButtons(to, body, buttons) {
  return axios.post(`https://graph.facebook.com/v20.0/${PHONE_NUMBER_ID}/messages`, {
    messaging_product: 'whatsapp',
    to,
    type: 'interactive',
    interactive: {
      type: 'button',
      body: { text: body },
      action: { buttons: buttons.map(b => ({ type: 'reply', reply: { id: b.id, title: b.title } })) }
    }
  }, { headers: { Authorization: `Bearer ${WHATSAPP_TOKEN}` } });
}

async function notifyOwner(text) {
  // Envoie une notif à ton propre numéro pour suivre les paiements
  if (!process.env.OWNER_WHATSAPP) return;
  try { await sendText(process.env.OWNER_WHATSAPP, text); } catch {}
}

async function askAI(userText, context) {
  if (!openai) return "Merci ! Dis-moi, on part sur Seul 10 000F ou Boutique 25 000F ?";
  try {
    const c = await openai.chat.completions.create({
      model: 'gpt-4o-mini',
      messages: [
        { role: 'system', content: `Tu es Geskap, vendeur auto WhatsApp Congo. Concis, chaleureux. Objectif closer 10k solo / 25k boutique. Paiement par MTN MoMo *105# et Orange *144# manuel. Finis par question fermée. Contexte: ${JSON.stringify(context)}` },
        { role: 'user', content: userText }
      ],
      max_tokens: 160
    });
    return c.choices[0].message.content;
  } catch { return "Top ! On part sur Seul (10 000F) ou Boutique (25 000F) ?"; }
}

async function logToSheet(data) {
  if (!process.env.GOOGLE_SHEET_WEBHOOK_URL) return;
  try { await axios.post(process.env.GOOGLE_SHEET_WEBHOOK_URL, data); } catch {}
}

function delay(ms) { return new Promise(r => setTimeout(r, ms)); }

app.get('/', (req, res) => res.send('Geskap Congo Bot ⚡ - Mode Manuel + MTN API actif'));
app.listen(process.env.PORT || 3000, () => console.log('Geskap CONGO running'));
