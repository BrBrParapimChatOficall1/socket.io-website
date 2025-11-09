// server.js
const express = require('express');
const http = require('http');
const socketIo = require('socket.io');
const path = require('path');

const app = express();
const server = http.createServer(app);
const io = socketIo(server);

// Хранилище данных (потом заменим на базу данных)
const users = new Map();
const messages = [];

// Статика для клиента
app.use(express.static(path.join(__dirname, 'public')));

// Обработка подключений
io.on('connection', (socket) => {
  console.log('👤 Новый пользователь подключился:', socket.id);

  // Регистрация пользователя
  socket.on('user_join', (username) => {
    users.set(socket.id, {
      id: socket.id,
      username: username,
      online: true
    });
    
    console.log(`✅ ${username} присоединился к чату`);
    
    // Уведомляем всех о новом пользователе
    socket.broadcast.emit('user_joined', username);
    
    // Отправляем историю сообщений новому пользователю
    socket.emit('message_history', messages);
    
    // Обновляем список онлайн пользователей
    updateOnlineUsers();
  });

  // Получение нового сообщения
  socket.on('send_message', (data) => {
    const user = users.get(socket.id);
    if (!user) return;

    const message = {
      id: Date.now().toString(),
      username: user.username,
      text: data.text,
      timestamp: new Date().toLocaleTimeString()
    };

    messages.push(message);
    
    // Рассылаем сообщение всем подключенным клиентам
    io.emit('new_message', message);
    console.log(`💬 ${user.username}: ${data.text}`);
  });

  // Обработка отключения
  socket.on('disconnect', () => {
    const user = users.get(socket.id);
    if (user) {
      console.log(`❌ ${user.username} отключился`);
      users.delete(socket.id);
      updateOnlineUsers();
    }
  });

  function updateOnlineUsers() {
    const onlineUsers = Array.from(users.values());
    io.emit('online_users', onlineUsers);
  }
});

const PORT = process.env.PORT || 3000;
server.listen(PORT, () => {
  console.log(`🚀 Сервер запущен на порту ${PORT}`);
  console.log(`📱 Откройте http://localhost:${PORT} в браузере`);
});
