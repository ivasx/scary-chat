// UI контролер для адмін панелі
class AdminUI {
  constructor(adminCore) {
    this.core = adminCore;
    this.currentMedia = null;
    this.init();
  }

  init() {
    this.setupEventListeners();
    this.setupSwipeHandlers();
  }

  setupEventListeners() {
    // Обробка натискань клавіш
    document.addEventListener('keydown', (e) => {
      if (e.key === 'Enter' && e.ctrlKey) {
        this.sendMessage();
      }
      if (e.key === 'Escape') {
        this.cancelReply();
        this.closeEditModal();
      }
    });

    // Автоматичне розширення textarea
    const messageTextarea = document.getElementById('adminMessage');
    if (messageTextarea) {
      messageTextarea.addEventListener('input', (e) => {
        e.target.style.height = 'auto';
        e.target.style.height = Math.min(e.target.scrollHeight, 120) + 'px';
        
        // Показати кнопку відправки якщо є текст
        const sendBtn = document.querySelector('.btn-success');
        if (sendBtn) {
          sendBtn.style.opacity = e.target.value.trim() ? '1' : '0.7';
        }
      });
    }
  }

  setupSwipeHandlers() {
    let startX = 0;
    let currentX = 0;
    let isDragging = false;
    let currentMessage = null;

    document.addEventListener('touchstart', (e) => {
      const messageEl = e.target.closest('.admin-message');
      if (messageEl && !messageEl.classList.contains('user')) {
        startX = e.touches[0].clientX;
        currentMessage = messageEl;
        isDragging = true;
      }
    });

    document.addEventListener('touchmove', (e) => {
      if (!isDragging || !currentMessage) return;
      
      e.preventDefault();
      currentX = e.touches[0].clientX;
      const diffX = currentX - startX;
      
      if (diffX > 0 && diffX < 100) {
        currentMessage.style.transform = `translateX(${diffX}px)`;
        currentMessage.style.opacity = 1 - (diffX / 200);
      }
    });

    document.addEventListener('touchend', (e) => {
      if (!isDragging || !currentMessage) return;
      
      const diffX = currentX - startX;
      
      if (diffX > 50) {
        // Активувати відповідь
        const messageId = currentMessage.dataset.messageId;
        this.setReplyTo(messageId);
      }
      
      // Скинути стилі
      currentMessage.style.transform = '';
      currentMessage.style.opacity = '';
      
      isDragging = false;
      currentMessage = null;
      startX = 0;
      currentX = 0;
    });
  }

  // === CHARACTERS UI ===
  renderCharactersList() {
    const container = document.getElementById('charactersList');
    if (!container) return;

    container.innerHTML = '';
    
    Array.from(this.core.characters.values()).forEach(character => {
      const item = document.createElement('div');
      item.className = `character-item ${this.core.currentCharacter?.id === character.id ? 'active' : ''}`;
      item.onclick = () => this.selectCharacter(character.id);
      
      const avatarHtml = character.avatar.startsWith('http') 
        ? `<img src="${character.avatar}" alt="${character.name}">` 
        : character.avatar;
      
      item.innerHTML = `
        <div class="character-avatar" style="background-color: ${character.color}">
          ${avatarHtml}
        </div>
        <div class="character-info">
          <div class="character-name">${character.name}</div>
          <div class="character-type">${this.getTypeLabel(character.type)}</div>
        </div>
      `;
      
      container.appendChild(item);
    });
  }

  updateCharacterForm() {
    if (!this.core.currentCharacter) return;
    
    const char = this.core.currentCharacter;
    document.getElementById('charName').value = char.name || '';
    document.getElementById('charAvatar').value = char.avatar || '';
    document.getElementById('charColor').value = char.color || '#ff4757';
    document.getElementById('charType').value = char.type || 'ghost';
    
    this.updateAvatarPreview(char.avatar);
  }

  clearCharacterForm() {
    document.getElementById('charName').value = '';
    document.getElementById('charAvatar').value = '';
    document.getElementById('charColor').value = '#ff4757';
    document.getElementById('charType').value = 'ghost';
    document.getElementById('avatarPreview').innerHTML = '👻';
    document.getElementById('avatarPreview').style.backgroundImage = '';
  }

  updateAvatarPreview(avatar) {
    const preview = document.getElementById('avatarPreview');
    if (!preview) return;
    
    if (avatar && avatar.startsWith('http')) {
      preview.innerHTML = `<img src="${avatar}" alt="Avatar">`;
      preview.style.backgroundImage = '';
    } else {
      preview.innerHTML = avatar || '👻';
      preview.style.backgroundImage = '';
    }
  }

  // === MESSAGES UI ===
  renderMessages() {
    const container = document.getElementById('messagesContainer');
    if (!container) return;

    container.innerHTML = '';
    
    const sortedMessages = Array.from(this.core.messages.values())
      .sort((a, b) => a.timestamp - b.timestamp);
    
    sortedMessages.forEach(message => {
      const messageEl = this.createMessageElement(message);
      container.appendChild(messageEl);
    });
    
    container.scrollTop = container.scrollHeight;
  }

  createMessageElement(message) {
    const div = document.createElement('div');
    div.className = `admin-message ${message.type || 'other'}`;
    div.dataset.messageId = message.id;
    
    if (this.core.editingMessage?.id === message.id) {
      div.classList.add('editing');
    }
    
    const avatarHtml = message.senderAvatar?.startsWith('http') 
      ? `<img src="${message.senderAvatar}" alt="${message.sender}" style="width: 100%; height: 100%; object-fit: cover; border-radius: 50%;">` 
      : message.senderAvatar || '👻';
    
    const replyHtml = message.replyTo ? this.createReplyHtml(message.replyTo) : '';
    const mediaHtml = message.media ? this.createMediaHtml(message.media) : '';
    const editedLabel = message.edited ? '<span style="font-size: 11px; opacity: 0.7;">(ред.)</span>' : '';
    
    div.innerHTML = `
      <div style="display: flex; align-items: flex-start; gap: 8px;">
        <div class="character-avatar" style="background-color: ${message.senderColor}; width: 32px; height: 32px; font-size: 14px;">
          ${avatarHtml}
        </div>
        <div style="flex: 1;">
          <div style="font-weight: 500; font-size: 13px; color: ${message.senderColor}; margin-bottom: 2px;">
            ${message.sender}
          </div>
          ${replyHtml}
          ${mediaHtml}
          <div style="margin-bottom: 4px;">${message.text}</div>
          <div style="font-size: 11px; opacity: 0.6;">
            ${this.core.formatTime(message.timestamp)} ${editedLabel}
          </div>
        </div>
      </div>
      <div class="message-controls">
        <button class="message-control-btn" onclick="adminCore.setReplyTo('${message.id}')" title="Відповісти">↩</button>
        <button class="message-control-btn" onclick="adminCore.startEditingMessage('${message.id}')" title="Редагувати">✏</button>
        <button class="message-control-btn" onclick="adminCore.deleteMessage('${message.id}')" title="Видалити">🗑</button>
      </div>
    `;
    
    return div;
  }

  createReplyHtml(replyTo) {
    const replyMessage = this.core.messages.get(replyTo) || replyTo;
    if (!replyMessage) return '';
    
    return `
      <div style="background: rgba(255,255,255,0.1); border-left: 2px solid #007aff; padding: 6px 8px; margin-bottom: 6px; border-radius: 4px; font-size: 12px;">
        <div style="color: #007aff; font-weight: 500; margin-bottom: 2px;">${replyMessage.sender || 'Користувач'}</div>
        <div style="opacity: 0.8;">${(replyMessage.text || '').substring(0, 50)}${replyMessage.text?.length > 50 ? '...' : ''}</div>
      </div>
    `;
  }

  createMediaHtml(media) {
    if (!media) return '';
    
    if (media.type === 'photo') {
      return `<img src="${media.url}" alt="Фото" style="max-width: 200px; max-height: 200px; border-radius: 8px; margin: 4px 0;">`;
    } else if (media.type === 'video') {
      return `<video src="${media.url}" controls style="max-width: 200px; max-height: 200px; border-radius: 8px; margin: 4px 0;"></video>`;
    }
    
    return '';
  }

  clearMessageInput() {
    const textarea = document.getElementById('adminMessage');
    if (textarea) {
      textarea.value = '';
      textarea.style.height = 'auto';
    }
    
    const mediaPreview = document.getElementById('mediaPreview');
    if (mediaPreview) {
      mediaPreview.innerHTML = '';
    }
    
    this.currentMedia = null;
  }

  // === REPLY UI ===
  showReplyPreview() {
    const preview = document.getElementById('replyPreview');
    const senderEl = document.getElementById('replyToUser');
    const textEl = document.getElementById('replyToText');
    
    if (preview && this.core.replyingTo) {
      senderEl.textContent = this.core.replyingTo.sender || 'Користувач';
      textEl.textContent = (this.core.replyingTo.text || '').substring(0, 50) + 
                          (this.core.replyingTo.text?.length > 50 ? '...' : '');
      preview.style.display = 'block';
    }
  }

  hideReplyPreview() {
    const preview = document.getElementById('replyPreview');
    if (preview) {
      preview.style.display = 'none';
    }
  }

  // === EDIT MODAL ===
  showEditModal(message) {
    const overlay = document.getElementById('editMessageOverlay');
    const textarea = document.getElementById('editMessageText');
    
    if (overlay && textarea) {
      textarea.value = message.text || '';
      overlay.style.display = 'flex';
      textarea.focus();
    }
  }

  closeEditModal() {
    const overlay = document.getElementById('editMessageOverlay');
    if (overlay) {
      overlay.style.display = 'none';
    }
    this.core.editingMessage = null;
    this.renderMessages();
  }

  // === USERS UI ===
  renderUsersList() {
    const container = document.getElementById('usersList');
    if (!container) return;

    // Отримання користувачів з бази даних
    const users = this.core.getActiveUsers();

    container.innerHTML = users.map(user => `
      <div class="user-item">
        <div>
          <div style="font-weight: 500;">${user.name}</div>
          <div class="user-status">
            <div class="status-indicator ${user.online ? 'online' : ''} ${this.core.mutedUsers.has(user.id) ? 'muted' : ''}"></div>
            <span>${user.online ? 'Онлайн' : 'Офлайн'} ${this.core.mutedUsers.has(user.id) ? '(заглушений)' : ''}</span>
          </div>
        </div>
        <div>
          <button class="btn ${this.core.mutedUsers.has(user.id) ? 'btn-success' : 'btn-warning'} btn-sm" 
                  onclick="adminUI.toggleMute('${user.id}')">
            ${this.core.mutedUsers.has(user.id) ? '🔊 Розглушити' : '🔇 Заглушити'}
          </button>
        </div>
      </div>
    `).join('');
  }

  // === STATISTICS ===
  updateStatistics() {
    const messagesCount = document.getElementById('messagesCount');
    const charactersCount = document.getElementById('charactersCount');
    const onlineCount = document.getElementById('onlineCount');
    
    if (messagesCount) messagesCount.textContent = this.core.messages.size;
    if (charactersCount) charactersCount.textContent = this.core.characters.size;
    if (onlineCount) onlineCount.textContent = this.core.getOnlineUsersCount();
  }

  // === UTILITY METHODS ===
  getTypeLabel(type) {
    const types = {
      ghost: 'Привид',
      demon: 'Демон',
      spirit: 'Дух',
      entity: 'Сутність',
      other: 'Інше'
    };
    return types[type] || 'Невідомо';
  }

  // === PUBLIC METHODS FOR GLOBAL ACCESS ===
  selectCharacter(characterId) {
    this.core.selectCharacter(characterId);
    this.renderCharactersList();
    this.updateCharacterForm();
  }

  setReplyTo(messageId) {
    this.core.setReplyTo(messageId);
    this.showReplyPreview();
  }

  cancelReply() {
    this.core.cancelReply();
    this.hideReplyPreview();
  }

  async toggleMute(userId) {
    const isMuted = this.core.mutedUsers.has(userId);
    await this.core.muteUser(userId, !isMuted);
    this.renderUsersList();
  }

  async sendMessage() {
    const textarea = document.getElementById('adminMessage');
    if (!textarea || !this.core.currentCharacter) return;

    const text = textarea.value.trim();
    const media = this.currentMedia;
    const replyTo = this.core.replyingTo?.id;

    if (!text && !media) return;

    const success = await this.core.sendMessage(text, media, replyTo);
    if (success) {
      this.clearMessageInput();
    }
  }

  async saveEditedMessage() {
    const textarea = document.getElementById('editMessageText');
    if (!textarea || !this.core.editingMessage) return;

    const newText = textarea.value.trim();
    if (!newText) return;

    const success = await this.core.editMessage(this.core.editingMessage.id, newText);
    if (success) {
      this.closeEditModal();
    }
  }

  insertQuickResponse(text) {
    const textarea = document.getElementById('adminMessage');
    if (textarea) {
      textarea.value = text;
      textarea.focus();
      textarea.dispatchEvent(new Event('input'));
    }
  }

  simulateTyping() {
    // Показати індикатор набору в основному чаті
    this.core.showTypingIndicator();
  }

  async clearChat() {
    await this.core.clearAllMessages();
  }
}

// Ініціалізація після завантаження adminCore
document.addEventListener('DOMContentLoaded', () => {
  if (window.adminCore) {
    window.adminUI = new AdminUI(window.adminCore);
    
    // Оновлення методів в adminCore для UI
    window.adminCore.renderCharactersList = () => window.adminUI.renderCharactersList();
    window.adminCore.renderMessages = () => window.adminUI.renderMessages();
    window.adminCore.renderUsersList = () => window.adminUI.renderUsersList();
    window.adminCore.updateCharacterForm = () => window.adminUI.updateCharacterForm();
    window.adminCore.clearCharacterForm = () => window.adminUI.clearCharacterForm();
    window.adminCore.clearMessageInput = () => window.adminUI.clearMessageInput();
    window.adminCore.showReplyPreview = () => window.adminUI.showReplyPreview();
    window.adminCore.hideReplyPreview = () => window.adminUI.hideReplyPreview();
    window.adminCore.showEditModal = (msg) => window.adminUI.showEditModal(msg);
    window.adminCore.updateUI = () => {
      window.adminUI.renderCharactersList();
      window.adminUI.renderMessages();
      window.adminUI.renderUsersList();
      window.adminUI.updateStatistics();
    };

    // Початкове завантаження UI
    window.adminCore.updateUI();
    
    console.log('AdminUI ініціалізовано успішно');
  } else {
    console.error('AdminCore не знайдено. Переконайтеся, що adminCore.js завантажено першим.');
  }
});

// Експорт для використання в інших модулях
if (typeof module !== 'undefined' && module.exports) {
  module.exports = AdminUI;
}