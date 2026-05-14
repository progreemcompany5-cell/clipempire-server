const express = require('express');
const multer = require('multer');
const axios = require('axios');
const FormData = require('form-data');
const fs = require('fs');
const path = require('path');
const cors = require('cors');

const app = express();
const PORT = process.env.PORT || 3000;

// ===== CONFIG =====
const META_USER_TOKEN = process.env.META_USER_TOKEN || 'EAAOE09U1WHgBRXOE6A2w2HUsF3SpvicUnBqsJdRIfc8A7sEZCJUvDZAYgqwfuxxbE0DEXE1wJN4q3BmwlpELhRg6KTBKkgmKpnd6Du8vNhZBCwosBZBxCVxPLN5MrNwS1x7f33oJ4VChelvna3RnAfgpZCxprCPyOLJr0ba2fACdEHnENmwTPwosRjJIvYfOBZB2GcMJZCZC8b8ZD';
const META_APP_ID = process.env.META_APP_ID || '990470280140920';
const META_APP_SECRET = process.env.META_APP_SECRET || '74752d10a3e43220556a3cc8614d61d0';
const GRAPH_API = 'https://graph.facebook.com/v19.0';

// ===== MIDDLEWARE =====
app.use(cors({ origin: '*' }));
app.use(express.json());

// File upload — store in /tmp
const upload = multer({
  dest: '/tmp/clipempire/',
  limits: { fileSize: 100 * 1024 * 1024 } // 100MB max
});

// Serve uploaded files publicly (needed for Instagram API)
app.use('/media', express.static('/tmp/clipempire'));

// ===== HEALTH CHECK =====
app.get('/', (req, res) => {
  res.json({ status: 'ClipEmpire Server running', version: '1.0.0' });
});

// ===== GET PAGE INFO =====
app.get('/pages', async (req, res) => {
  try {
    const response = await axios.get(`${GRAPH_API}/me/accounts`, {
      params: { access_token: META_USER_TOKEN }
    });
    res.json({ success: true, pages: response.data.data });
  } catch (err) {
    res.status(500).json({ success: false, error: err.response?.data || err.message });
  }
});

// ===== UPLOAD FILE & GET PUBLIC URL =====
app.post('/upload', upload.single('file'), (req, res) => {
  if (!req.file) return res.status(400).json({ success: false, error: 'No file uploaded' });
  const publicUrl = `${req.protocol}://${req.get('host')}/media/${req.file.filename}`;
  // Schedule deletion after 1 hour
  setTimeout(() => {
    try { fs.unlinkSync(req.file.path); } catch(e) {}
  }, 3600000);
  res.json({ success: true, url: publicUrl, filename: req.file.filename, originalName: req.file.originalname });
});

// ===== POST IMAGE TO INSTAGRAM =====
app.post('/instagram/image', upload.single('file'), async (req, res) => {
  try {
    const { caption, page_id } = req.body;
    if (!req.file) return res.status(400).json({ success: false, error: 'No image file' });

    // Get public URL for this server
    const serverUrl = `${req.protocol}://${req.get('host')}`;
    const imageUrl = `${serverUrl}/media/${req.file.filename}`;

    // Get Instagram account ID
    const pageRes = await axios.get(`${GRAPH_API}/${page_id}`, {
      params: { fields: 'instagram_business_account', access_token: META_USER_TOKEN }
    });
    const igId = pageRes.data.instagram_business_account?.id;
    if (!igId) return res.status(400).json({ success: false, error: 'No Instagram business account connected to this page' });

    // Wait a moment for file to be accessible
    await new Promise(r => setTimeout(r, 1000));

    // Step 1: Create media container
    const containerRes = await axios.post(`${GRAPH_API}/${igId}/media`, null, {
      params: {
        image_url: imageUrl,
        caption: caption || '',
        access_token: META_USER_TOKEN
      }
    });
    const creationId = containerRes.data.id;

    // Step 2: Wait for processing
    await new Promise(r => setTimeout(r, 3000));

    // Step 3: Publish
    const publishRes = await axios.post(`${GRAPH_API}/${igId}/media_publish`, null, {
      params: {
        creation_id: creationId,
        access_token: META_USER_TOKEN
      }
    });

    // Cleanup file
    setTimeout(() => { try { fs.unlinkSync(req.file.path); } catch(e) {} }, 5000);

    res.json({ success: true, post_id: publishRes.data.id, platform: 'instagram', type: 'image' });
  } catch (err) {
    console.error('Instagram image error:', err.response?.data || err.message);
    res.status(500).json({ success: false, error: err.response?.data?.error?.message || err.message });
  }
});

// ===== POST VIDEO/REEL TO INSTAGRAM =====
app.post('/instagram/video', upload.single('file'), async (req, res) => {
  try {
    const { caption, page_id } = req.body;
    if (!req.file) return res.status(400).json({ success: false, error: 'No video file' });

    const serverUrl = `${req.protocol}://${req.get('host')}`;
    const videoUrl = `${serverUrl}/media/${req.file.filename}`;

    // Get Instagram account ID
    const pageRes = await axios.get(`${GRAPH_API}/${page_id}`, {
      params: { fields: 'instagram_business_account', access_token: META_USER_TOKEN }
    });
    const igId = pageRes.data.instagram_business_account?.id;
    if (!igId) return res.status(400).json({ success: false, error: 'No Instagram business account connected' });

    await new Promise(r => setTimeout(r, 2000));

    // Step 1: Create Reel container
    const containerRes = await axios.post(`${GRAPH_API}/${igId}/media`, null, {
      params: {
        media_type: 'REELS',
        video_url: videoUrl,
        caption: caption || '',
        share_to_feed: true,
        access_token: META_USER_TOKEN
      }
    });
    const creationId = containerRes.data.id;

    // Step 2: Poll until ready (video processing takes time)
    let ready = false;
    let attempts = 0;
    while (!ready && attempts < 20) {
      await new Promise(r => setTimeout(r, 5000));
      const statusRes = await axios.get(`${GRAPH_API}/${creationId}`, {
        params: { fields: 'status_code', access_token: META_USER_TOKEN }
      });
      if (statusRes.data.status_code === 'FINISHED') ready = true;
      if (statusRes.data.status_code === 'ERROR') throw new Error('Instagram video processing failed');
      attempts++;
    }

    // Step 3: Publish
    const publishRes = await axios.post(`${GRAPH_API}/${igId}/media_publish`, null, {
      params: { creation_id: creationId, access_token: META_USER_TOKEN }
    });

    setTimeout(() => { try { fs.unlinkSync(req.file.path); } catch(e) {} }, 5000);

    res.json({ success: true, post_id: publishRes.data.id, platform: 'instagram', type: 'reel' });
  } catch (err) {
    console.error('Instagram video error:', err.response?.data || err.message);
    res.status(500).json({ success: false, error: err.response?.data?.error?.message || err.message });
  }
});

// ===== POST IMAGE TO FACEBOOK =====
app.post('/facebook/image', upload.single('file'), async (req, res) => {
  try {
    const { caption, page_id } = req.body;
    if (!req.file) return res.status(400).json({ success: false, error: 'No image file' });

    // Get page access token
    const pagesRes = await axios.get(`${GRAPH_API}/me/accounts`, {
      params: { access_token: META_USER_TOKEN }
    });
    const page = pagesRes.data.data.find(p => p.id === page_id) || pagesRes.data.data[0];
    if (!page) return res.status(400).json({ success: false, error: 'No Facebook page found' });

    // Upload photo to Facebook
    const form = new FormData();
    form.append('source', fs.createReadStream(req.file.path), req.file.originalname);
    form.append('message', caption || '');
    form.append('access_token', page.access_token);

    const uploadRes = await axios.post(`${GRAPH_API}/${page.id}/photos`, form, {
      headers: form.getHeaders()
    });

    setTimeout(() => { try { fs.unlinkSync(req.file.path); } catch(e) {} }, 5000);

    res.json({ success: true, post_id: uploadRes.data.id, platform: 'facebook', type: 'image' });
  } catch (err) {
    console.error('Facebook image error:', err.response?.data || err.message);
    res.status(500).json({ success: false, error: err.response?.data?.error?.message || err.message });
  }
});

// ===== POST VIDEO TO FACEBOOK =====
app.post('/facebook/video', upload.single('file'), async (req, res) => {
  try {
    const { caption, page_id } = req.body;
    if (!req.file) return res.status(400).json({ success: false, error: 'No video file' });

    // Get page access token
    const pagesRes = await axios.get(`${GRAPH_API}/me/accounts`, {
      params: { access_token: META_USER_TOKEN }
    });
    const page = pagesRes.data.data.find(p => p.id === page_id) || pagesRes.data.data[0];
    if (!page) return res.status(400).json({ success: false, error: 'No Facebook page found' });

    // Upload video to Facebook
    const form = new FormData();
    form.append('source', fs.createReadStream(req.file.path), req.file.originalname);
    form.append('description', caption || '');
    form.append('access_token', page.access_token);

    const uploadRes = await axios.post(`${GRAPH_API}/${page.id}/videos`, form, {
      headers: { ...form.getHeaders() },
      maxBodyLength: Infinity,
      maxContentLength: Infinity
    });

    setTimeout(() => { try { fs.unlinkSync(req.file.path); } catch(e) {} }, 5000);

    res.json({ success: true, video_id: uploadRes.data.id, platform: 'facebook', type: 'video' });
  } catch (err) {
    console.error('Facebook video error:', err.response?.data || err.message);
    res.status(500).json({ success: false, error: err.response?.data?.error?.message || err.message });
  }
});

// ===== POST TO BOTH PLATFORMS =====
app.post('/post-all', upload.single('file'), async (req, res) => {
  const { caption_ig, caption_fb, page_id, platforms, type } = req.body;
  const results = { instagram: null, facebook: null };
  const errors = {};

  if (!req.file) return res.status(400).json({ success: false, error: 'No file uploaded' });

  const serverUrl = `${req.protocol}://${req.get('host')}`;
  const fileUrl = `${serverUrl}/media/${req.file.filename}`;
  const isVideo = req.file.mimetype.startsWith('video/');

  // Get pages
  let pageToken = META_USER_TOKEN;
  let igId = null;
  try {
    const pagesRes = await axios.get(`${GRAPH_API}/me/accounts`, { params: { access_token: META_USER_TOKEN } });
    const page = pagesRes.data.data.find(p => p.id === page_id) || pagesRes.data.data[0];
    if (page) pageToken = page.access_token;

    const pageRes = await axios.get(`${GRAPH_API}/${page?.id || page_id}`, {
      params: { fields: 'instagram_business_account', access_token: pageToken }
    });
    igId = pageRes.data.instagram_business_account?.id;
  } catch(e) { console.log('Page fetch error:', e.message); }

  await new Promise(r => setTimeout(r, 1500));

  // Post to Facebook
  if (!platforms || platforms.includes('facebook')) {
    try {
      const form = new FormData();
      const endpoint = isVideo ? 'videos' : 'photos';
      form.append('source', fs.createReadStream(req.file.path), req.file.originalname);
      form.append(isVideo ? 'description' : 'message', caption_fb || caption_ig || '');
      form.append('access_token', pageToken);
      const fbRes = await axios.post(`${GRAPH_API}/${page_id}/` + endpoint, form, {
        headers: form.getHeaders(), maxBodyLength: Infinity, maxContentLength: Infinity
      });
      results.facebook = { success: true, id: fbRes.data.id };
    } catch(e) { errors.facebook = e.response?.data?.error?.message || e.message; }
  }

  // Post to Instagram
  if ((!platforms || platforms.includes('instagram')) && igId) {
    try {
      let containerRes;
      if (isVideo) {
        containerRes = await axios.post(`${GRAPH_API}/${igId}/media`, null, {
          params: { media_type: 'REELS', video_url: fileUrl, caption: caption_ig || '', share_to_feed: true, access_token: META_USER_TOKEN }
        });
        // Wait for processing
        let ready = false, attempts = 0;
        while (!ready && attempts < 15) {
          await new Promise(r => setTimeout(r, 5000));
          const s = await axios.get(`${GRAPH_API}/${containerRes.data.id}`, { params: { fields: 'status_code', access_token: META_USER_TOKEN } });
          if (s.data.status_code === 'FINISHED') ready = true;
          attempts++;
        }
      } else {
        containerRes = await axios.post(`${GRAPH_API}/${igId}/media`, null, {
          params: { image_url: fileUrl, caption: caption_ig || '', access_token: META_USER_TOKEN }
        });
        await new Promise(r => setTimeout(r, 3000));
      }
      const pubRes = await axios.post(`${GRAPH_API}/${igId}/media_publish`, null, {
        params: { creation_id: containerRes.data.id, access_token: META_USER_TOKEN }
      });
      results.instagram = { success: true, id: pubRes.data.id };
    } catch(e) { errors.instagram = e.response?.data?.error?.message || e.message; }
  }

  setTimeout(() => { try { fs.unlinkSync(req.file.path); } catch(e) {} }, 10000);

  const anySuccess = results.facebook?.success || results.instagram?.success;
  res.json({ success: anySuccess, results, errors });
});

// ===== START SERVER =====
// Ensure temp dir exists
if (!fs.existsSync('/tmp/clipempire')) fs.mkdirSync('/tmp/clipempire', { recursive: true });

app.listen(PORT, () => {
  console.log(`ClipEmpire server running on port ${PORT}`);
});
