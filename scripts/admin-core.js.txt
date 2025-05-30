import { db, storage } from './firebase-config.js';
import { ref, push, set, get, child, onValue, off, remove, update } from "https://www.gstatic.com/firebasejs/10.12.0/firebase-database.js";
import { ref as storageRef, uploadBytes, getDownloadURL, deleteObject } from "https://www.gstatic.com/firebasejs/10.12.0/firebase-storage.js";

class AdminCore {
  constructor() {
    this.currentCharacter = null;
    this.characters = new Map();
    this.messages = new Map();
    this.replyingTo = null;
    this.editingMessage = null;
    this.mutedUsers = new Set();
    this.settings = {
      chatTitle: 'Мертві душі',
      background: 'gradient1',
      colorScheme: 'default'
    };
    
    this.init();
  }

  async init() {
    await this.loadCharacters();
    await this.loadMessages();
    await this.loadSettings();
    this.setupRealtimeListeners();
    this.updateUI();
    this.updateStatistics();
  }

  // === CHARACTERS MANAGEMENT ===
  async loadCharacters() {
    try {
      const snapshot = await get(child(ref(db), 'characters'));
      if (snapshot.exists()) {
        const data = snapshot.val();
        this.characters.clear();
        Object.entries(data).forEach(([id, character]) => {
          this.characters.set(id, { id, ...character });
        });
      }
      this.renderCharactersList();
    } catch (error) {
      console.error('Помилка завантаження персонажів:', error);
    }
  }

  async saveCharacter(characterData) {
    try {
      if (characterData.id) {
        // Оновлення існуючого персонажа
        await set(ref(db, `characters/${characterData.id}`), {
          name: characterData.name,
          avatar: characterData.avatar,
          color: characterData.color,
          type: characterData.type,
          updatedAt: Date.now()
        });
        this.characters.set(characterData.id, characterData);
      } else {
        // Створення нового персонажа
        const newRef = push(ref(db, 'characters'));
        const newCharacter = {
          name: characterData.name,
          avatar: characterData.avatar,
          color: characterData.color,
          type: characterData.type,
          createdAt: Date.now()
        };
        await set(newRef, newCharacter);
        this.characters.set(newRef.key, { id: newRef.key, ...newCharacter });
      }
      
      this.renderCharactersList();
      this.updateStatistics();
      return true;
    } catch (error) {
      console.error('Помилка збереження персонажа:', error);
      return false;
    }
  }

  async deleteCharacter(characterId) {
    if (!characterId || !confirm('Ви впевнені, що хочете видалити цього персонажа?')) {
      return false;
    }

    try {
      await remove(ref(db, `characters/${characterId}`));
      this.characters.delete(characterId);
      
      if (this.currentCharacter?.id === characterId) {
        this.currentCharacter = null;
        this.clearCharacterForm();
      }
      
      this.renderCharactersList();
      this.updateStatistics();
      return true;
    } catch (error) {
      console.error('Помилка видалення персонажа:', error);
      return false;
    }
  }

  // === FILE UPLOAD ===
  async uploadFile(file, folder = 'media') {
    try {
      const filename = `${folder}/${Date.now()}_${file.name}`;
      const fileRef = storageRef(storage, filename);
      const snapshot = await uploadBytes(fileRef, file);
      const downloadURL = await getDownloadURL(snapshot.ref);
      return downloadURL;
    } catch (error) {
      console.error('Помилка завантаження файлу:', error);
      throw error;
    }
  }

  // === MESSAGES MANAGEMENT ===
  async loadMessages() {
    try {
      const snapshot = await get(child(ref(db), 'messages'));
      if (snapshot.exists()) {
        const data = snapshot.val();
        this.messages.clear();
        Object.entries(data).forEach(([id, message]) => {
          this.messages.set(id, { id, ...message });
        });
      }
      this.renderMessages();
    } catch (error) {
      console.error('Помилка завантаження повідомлень:', error);
    }
  }

  async sendMessage(text, media = null, replyTo = null) {
    if (!this.currentCharacter || (!text.trim() && !media)) {
      return false;
    }

    try {
      const message = {
        text: text.trim(),
        sender: this.currentCharacter.name,
        senderId: this.currentCharacter.id,
        senderAvatar: this.currentCharacter.avatar,
        senderColor: this.currentCharacter.color,
        timestamp: Date.now(),
        type: 'admin',
        media: media,
        replyTo: replyTo
      };

      const newRef = push(ref(db, 'messages'));
      await set(newRef, message);
      
      this.messages.set(newRef.key, { id: newRef.key, ...message });
      this.renderMessages();
      this.clearMessageInput();
      this.cancelReply();
      this.updateStatistics();
      
      return true;
    } catch (error) {
      console.error('Помилка відправки повідомлення:', error);
      return false;
    }
  }

  async editMessage(messageId, newText) {
    try {
      const message = this.messages.get(messageId);
      if (!message) return false;

      const updates = {
        text: newText,
        edited: true,
        editedAt: Date.now()
      };

      await update(ref(db, `messages/${messageId}`), updates);
      
      Object.assign(message, updates);
      this.renderMessages();
      return true;
    } catch (error) {
      console.error('Помилка редагування повідомлення:', error);
      return false;
    }
  }

  async deleteMessage(messageId) {
    if (!confirm('Ви впевнені, що хочете видалити це повідомлення?')) {
      return false;
    }

    try {
      await remove(ref(db, `messages/${messageId}`));
      this.messages.delete(messageId);
      this.renderMessages();
      this.updateStatistics();
      return true;
    } catch (error) {
      console.error('Помилка видалення повідомлення:', error);
      return false;
    }
  }

  async clearAllMessages() {
    if (!confirm('Ви впевнені, що хочете очистити весь чат? Цю дію неможливо скасувати.')) {
      return false;
    }

    try {
      await remove(ref(db, 'messages'));
      this.messages.clear();
      this.renderMessages();
      this.updateStatistics();
      return true;
    } catch (error) {
      console.error('Помилка очищення чату:', error);
      return false;
    }
  }

  // === USER MANAGEMENT ===
  async muteUser(userId, mute = true) {
    try {
      if (mute) {
        this.mutedUsers.add(userId);
      } else {
        this.mutedUsers.delete(userId);
      }
      
      await set(ref(db, `mutedUsers/${userId}`), mute ? true : null);
      this.renderUsersList();
      return true;
    } catch (error) {
      console.error('Помилка зміни статусу користувача:', error);
      return false;
    }
  }

  // === SETTINGS MANAGEMENT ===
  async loadSettings() {
    try {
      const snapshot = await get(child(ref(db), 'settings'));
      if (snapshot.exists()) {
        this.settings = { ...this.settings, ...snapshot.val() };
      }
      this.applySettings();
    } catch (error) {
      console.error('Помилка завантаження налаштувань:', error);
    }
  }

  async saveSettings(newSettings) {
    try {
      this.settings = { ...this.settings, ...newSettings };
      await set(ref(db, 'settings'), this.settings);
      this.applySettings();
      return true;
    } catch (error) {
      console.error('Помилка збереження налаштувань:', error);
      return false;
    }
  }

  applySettings() {
    // Застосування налаштувань до інтерфейсу
    if (this.settings.chatTitle) {
      document.title = `Адмін панель - ${this.settings.chatTitle}`;
    }
    
    // Фон та кольорова схема будуть застосовані в admin-settings.js
  }

  // === REALTIME LISTENERS ===
  setupRealtimeListeners() {
    // Слухач повідомлень
    onValue(ref(db, 'messages'), (snapshot) => {
      if (snapshot.exists()) {
        const data = snapshot.val();
        this.messages.clear();
        Object.entries(data).forEach(([id, message]) => {
          this.messages.set(id, { id, ...message });
        });
        this.renderMessages();
        this.updateStatistics();
      }
    });

    // Слухач персонажів
    onValue(ref(db, 'characters'), (snapshot) => {
      if (snapshot.exists()) {
        const data = snapshot.val();
        this.characters.clear();
        Object.entries(data).forEach(([id, character]) => {
          this.characters.set(id, { id, ...character });
        });
        this.renderCharactersList();
        this.updateStatistics();
      }
    });
  }

  // === UI METHODS ===
  selectCharacter(characterId) {
    this.currentCharacter = this.characters.get(characterId);
    this.updateCharacterForm();
    this.renderCharactersList();
  }

  setReplyTo(messageId) {
    const message = this.messages.get(messageId);
    if (message) {
      this.replyingTo = message;
      this.showReplyPreview();
    }
  }

  cancelReply() {
    this.replyingTo = null;
    this.hideReplyPreview();
  }

  startEditingMessage(messageId) {
    const message = this.messages.get(messageId);
    if (message) {
      this.editingMessage = message;
      this.showEditModal(message);
    }
  }

  // === UTILITY METHODS ===
  formatTime(timestamp) {
    const date = new Date(timestamp);
    return date.toLocaleTimeString('uk-UA', { 
      hour12: false, 
      hour: '2-digit', 
      minute: '2-digit' 
    });
  }

  generateId() {
    return Date.now().toString() + Math.random().toString(36).substr(2, 5);
  }

  // === UI RENDERING METHODS (будуть імплементовані в admin-ui.js) ===
  renderCharactersList() { /* Імплементація в admin-ui.js */ }
  renderMessages() { /* Імплементація в admin-ui.js */ }
  renderUsersList() { /* Імплементація в admin-ui.js */ }
  updateCharacterForm() { /* Імплементація в admin-ui.js */ }
  clearCharacterForm() { /* Імплементація в admin-ui.js */ }
  clearMessageInput() { /* Імплементація в admin-ui.js */ }
  showReplyPreview() { /* Імплементація в admin-ui.js */ }
  hideReplyPreview() { /* Імплементація в admin-ui.js */ }
  showEditModal() { /* Імплементація в admin-ui.js */ }
  updateUI() { /* Імплементація в admin-ui.js */ }
  updateStatistics() { /* Імплементація в admin-ui.js */ }
}

// Глобальний екземпляр
window.adminCore = new AdminCore();

export default AdminCore;