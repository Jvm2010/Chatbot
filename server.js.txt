const express = require('express');
const http = require('http');
const socketIo = require('socket.io');
const cors = require('cors');
const jwt = require('jsonwebtoken');
const bcrypt = require('bcrypt');
const { createClient } = require('@supabase/supabase-js');

const app = express();
const server = http.createServer(app);
const io = socketIo(server, { cors: { origin: '*' } });

app.use(cors());
app.use(express.json());

// Conectando ao Supabase
const supabase = createClient(
  'https://texlojbgwyvsdrkwqhgl.supabase.co',
  'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6InRleGxvamJnd3l2c2Rya3dxaGdsIiwicm9sZSI6ImFub24iLCJpYXQiOjE3NDU4NjIwOTEsImV4cCI6MjA2MTQzODA5MX0.dTKA8qRcJIR5WiYvEDW8nWgNHipYgHFtD-8vMBO04mM'
);

// Criar usuário admin se não existir
async function createDefaultUser() {
  const { data, error } = await supabase
    .from('users')
    .select('*')
    .eq('username', 'admin')
    .single();

  if (!data) {
    const hashedPassword = await bcrypt.hash('admin123', 10);
    await supabase.from('users').insert([
      { username: 'admin', password: hashedPassword }
    ]);
    console.log('Usuário admin criado.');
  }
}
createDefaultUser();

// Login
app.post('/api/login', async (req, res) => {
  const { username, password } = req.body;

  const { data: user } = await supabase
    .from('users')
    .select('*')
    .eq('username', username)
    .single();

  if (!user || !(await bcrypt.compare(password, user.password))) {
    return res.status(401).send('Credenciais inválidas');
  }

  const token = jwt.sign({ id: user.id }, 'secret', { expiresIn: '8h' });
  res.json({ token });
});

// Listar conversas esperando atendimento
app.get('/api/chats', async (req, res) => {
  const { data: chats } = await supabase
    .from('chats')
    .select('*')
    .eq('status', 'waiting');

  res.json(chats || []);
});

// Websockets
io.on('connection', (socket) => {
  console.log('Novo cliente conectado');

  socket.on('userMessage', async (data) => {
    let { data: chat } = await supabase
      .from('chats')
      .select('*')
      .eq('userId', data.userId)
      .single();

    if (!chat) {
      await supabase.from('chats').insert([
        { userId: data.userId, messages: [data.message], status: 'waiting' }
      ]);
    } else {
      const newMessages = [...chat.messages, data.message];
      await supabase.from('chats').update({ messages: newMessages }).eq('id', chat.id);
    }

    io.emit('newMessage', data);
  });

  socket.on('attendantResponse', async (data) => {
    const { data: chat } = await supabase
      .from('chats')
      .select('*')
      .eq('id', data.chatId)
      .single();

    if (chat) {
      const newMessages = [...chat.messages, data.message];
      await supabase.from('chats').update({
        messages: newMessages,
        status: 'attending'
      }).eq('id', chat.id);

      io.emit('messageUpdated', data);
    }
  });

  socket.on('disconnect', () => {
    console.log('Cliente desconectado');
  });
});

const PORT = process.env.PORT || 4000;
server.listen(PORT, () => console.log(`Servidor rodando na porta ${PORT}`));
