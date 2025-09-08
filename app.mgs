import 'dotenv/config';
import express from 'express';
import cors from 'cors';
import helmet from 'helmet';
import morgan from 'morgan';
import { createClient } from '@supabase/supabase-js';

const app = express();
app.use(helmet());
app.use(express.json({ limit: '1mb' }));

// CORS
const origins = (process.env.CORS_ORIGINS || '*').split(',').map(s => s.trim());
const corsOptions = origins.includes('*') ? {} : {
  origin: function (origin, callback) {
    if (!origin || origins.indexOf(origin) !== -1) return callback(null, true);
    return callback(new Error('Not allowed by CORS'));
  },
  credentials: true
};
app.use(cors(corsOptions));

// Logging
app.use(morgan('tiny'));

// Supabase (service role)
const { SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY } = process.env;
if (!SUPABASE_URL || !SUPABASE_SERVICE_ROLE_KEY) {
  console.error('Missing SUPABASE_URL or SUPABASE_SERVICE_ROLE_KEY');
  process.exit(1);
}
const supabase = createClient(SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY);

// Health
app.get('/health', (req, res) => {
  res.json({ ok: true, time: new Date().toISOString(), service: 'sheeni-server', version: '1.0.0' });
});

// Create job
app.post('/jobs', async (req, res) => {
  try {
    const { name, phone, address, lat, lng, minutes, notes } = req.body || {};
    if (!name || !phone || !address || !minutes) {
      return res.status(400).json({ error: 'name, phone, address, minutes required' });
    }
    const pricePerHour = 45;
    const amountCents = Math.max(1, Math.round((Number(minutes) / 60) * pricePerHour * 100));
    const { data, error } = await supabase
      .from('jobs')
      .insert({
        status: 'REQUESTED', address, lat, lng, notes,
        minutes_booked: minutes, price_cents: amountCents,
        customer_name: name, customer_phone: phone
      })
      .select()
      .single();
    if (error) throw error;
    res.json({ ok: true, job: data });
  } catch (e) {
    console.error(e);
    res.status(500).json({ error: 'server_error', message: e.message });
  }
});

// Cleaner online
app.post('/cleaner/online', async (req, res) => {
  try {
    const { name, phone, lat, lng, online } = req.body || {};
    if (!name || !phone) return res.status(400).json({ error: 'name and phone required' });
    const { data, error } = await supabase
      .from('cleaners')
      .upsert({ user_phone: phone, name, online: !!online, lat, lng })
      .select()
      .single();
    if (error) throw error;
    res.json({ ok: true, cleaner: data });
  } catch (e) {
    console.error(e);
    res.status(500).json({ error: 'server_error', message: e.message });
  }
});

// Assign nearest (demo)
app.post('/jobs/:id/assign-nearest', async (req, res) => {
  try {
    const jobId = req.params.id;
    const { data: job, error: jobErr } = await supabase.from('jobs').select().eq('id', jobId).single();
    if (jobErr) throw jobErr;
    if (!job) return res.status(404).json({ error: 'job_not_found' });

    const { data: cleaners, error: cleanersErr } = await supabase.rpc('find_nearest_cleaner', { jlat: job.lat, jlng: job.lng });
    if (cleanersErr) throw cleanersErr;
    const nearest = cleaners && cleaners.length ? cleaners[0] : null;
    if (!nearest) return res.json({ ok: true, message: 'no_cleaners_online' });

    const { data: updated, error: updErr } = await supabase
      .from('jobs')
      .update({ assigned_cleaner_id: nearest.id, status: 'ACCEPTED' })
      .eq('id', job.id)
      .select()
      .single();
    if (updErr) throw updErr;

    res.json({ ok: true, job: updated, assigned_cleaner: nearest });
  } catch (e) {
    console.error(e);
    res.status(500).json({ error: 'server_error', message: e.message });
  }
});

// Waitlist endpoints
app.post('/waitlist/customer', async (req, res) => {
  try {
    const { name, email, phone, zip, notes } = req.body || {};
    if (!name || !email) return res.status(400).json({ error: 'name and email required' });
    const { data, error } = await supabase
      .from('waitlist_customers')
      .insert({ name, email, phone, zip, notes })
      .select()
      .single();
    if (error) throw error;
    res.json({ ok: true, entry: data });
  } catch (e) {
    console.error(e);
    res.status(500).json({ error: 'server_error', message: e.message });
  }
});
app.post('/waitlist/cleaner', async (req, res) => {
  try {
    const { name, email, phone, service_area, experience } = req.body || {};
    if (!name || !email) return res.status(400).json({ error: 'name and email required' });
    const { data, error } = await supabase
      .from('waitlist_cleaners')
      .insert({ name, email, phone, service_area, experience })
      .select()
      .single();
    if (error) throw error;
    res.json({ ok: true, entry: data });
  } catch (e) {
    console.error(e);
    res.status(500).json({ error: 'server_error', message: e.message });
  }
});

// 404
app.use((req, res) => res.status(404).json({ error: 'not_found' }));

const port = Number(process.env.PORT || 8080);
app.listen(port, () => console.log(`✅ Sheeni server listening on :${port}`));
