// Імпорт Firebase конфігурації
import { db, storage } from './firebase-config.js';
import { ref, push, onValue, serverTimestamp, off } from "https://www.gstatic.com/firebasejs/10.12.0/firebase-database.js";
import { ref as storageRef, uploadBytes, getDownloadURL } from "https://www.gstatic.com/firebasejs/10.12.0/firebase-storage.js";

// Глобальні змінні
let currentUser = null;
let messages = [];
let characters = {};
let chatSettings = {};
let isTyping = false;
let replyingTo = null;
let touchStartX = 0;
let touchStartY = 0;
let swipeThreshold = 50;

// DOM елементи
const messagesArea = document.getElementById('messagesArea');
const messageInput = document.getElementById('messageInput');
const sendButton = document.getElementById('sendButton');
const typingIndicator = document.getElementById('typingIndicator');
const replyPreview = document.getElementById('replyPreview');
const replySender = document.getElementById('replySender');
const replyText = document.getElementById('replyText');
const attachmentMenu = document.getElementById('attachmentMenu');
const photoInput = document.getElementById('photoInput');
const videoInput = document.getElementById('videoInput');
const mutedNotice = document.getElementById('mutedNotice');
const chatTitle = document.getElementById('chatTitle');
const chatAvatar = document.getElementById('chatAvatar');
const onlineStatus = document.getElementById('onlineStatus');

// Ініціалізація
document.addEventListener('DOMContentLoaded', () => {
    initializeApp();
    setupEventListeners();
    loadChatData();
});

// Ініціалізація додатка
function initializeApp() {
    // Генерація унікального ID користувача
    currentUser = localStorage.getItem('chatUserId') || generateUserId();
    localStorage.setItem('chatUserId', currentUser);
    
    // Встановлення початкових значень
    updateSendButtonVisibility();
    
    console.log('Чат ініціалізовано для користувача:', currentUser);
}

// Генерація ID користувача
function generateUserId() {
    return 'user_' + Math.random().toString(36).substr(2, 9) + '_' + Date.now();
}

// Налаштування слухачів подій
function setupEventListeners() {
    // Відправка повідомлення
    sendButton.addEventListener('click', sendMessage);
    messageInput.addEventListener('keypress', (e) => {
        if (e.key === 'Enter' && !e.shiftKey) {
            e.preventDefault();
            sendMessage();
        }
    });
    
    // Автоматичне розширення textarea
    messageInput.addEventListener('input', () => {
        messageInput.style.height = 'auto';
        messageInput.style.height = Math.min(messageInput.scrollHeight, 120) + 'px';
        updateSendButtonVisibility();
    });
    
    // Меню вкладень
    document.querySelector('.attachment-btn').addEventListener('click', toggleAttachmentMenu);
    
    // Завантаження файлів
    photoInput.addEventListener('change', (e) => handleFileUpload(e, 'photo'));
    videoInput.addEventListener('change', (e) => handleFileUpload(e, 'video'));
    
    // Закриття меню при кліку поза ним
    document.addEventListener('click', (e) => {
        if (!e.target.closest('.attachment-btn') && !e.target.closest('.attachment-menu')) {
            attachmentMenu.style.display = 'none';
        }
    });
    
    // Touch події для свайпів
    messagesArea.addEventListener('touchstart', handleTouchStart, { passive: true });
    messagesArea.addEventListener('touchmove', handleTouchMove, { passive: true });
    messagesArea.addEventListener('touchend', handleTouchEnd);
}

// Завантаження даних чату
function loadChatData() {
    // Завантаження повідомлень
    const messagesRef = ref(db, 'messages');
    onValue(messagesRef, (snapshot) => {
        const data = snapshot.val();
        messages = data ? Object.entries(data).map(([id, msg]) => ({ id, ...msg })) : [];
        messages.sort((a, b) => (a.timestamp || 0) - (b.timestamp || 0));
        renderMessages();
    });
    
    // Завантаження персонажів
    const charactersRef = ref(db, 'characters');
    onValue(charactersRef, (snapshot) => {
        characters = snapshot.val() || {};
    });
    
    // Завантаження налаштувань
    const settingsRef = ref(db, 'settings');
    onValue(settingsRef, (snapshot) => {
        chatSettings = snapshot.val() || {};
        applyChatSettings();
    });
    
    // Індикатор набору тексту
    const typingRef = ref(db, 'typing');
    onValue(typingRef, (snapshot) => {
        const typingData = snapshot.val();
        showTypingIndicator(typingData && Object.keys(typingData).length > 0);
    });
}

// Відправка повідомлення
async function sendMessage() {
    const text = messageInput.value.trim();
    if (!text && !replyingTo) return;
    
    const messageData = {
        text: text,
        sender: currentUser,
        timestamp: serverTimestamp(),
        type: 'user'
    };
    
    if (replyingTo) {
        messageData.replyTo = replyingTo;
        cancelReply();
    }
    
    try {
        await push(ref(db, 'messages'), messageData);
        messageInput.value = '';
        messageInput.style.height = 'auto';
        updateSendButtonVisibility();
        scrollToBottom();
    } catch (error) {
        console.error('Помилка відправки повідомлення:', error);
        showError('Не вдалося відправити повідомлення');
    }
}

// Завантаження файлу
async function handleFileUpload(event, type) {
    const file = event.target.files[0];
    if (!file) return;
    
    // Перевірка розміру файлу (макс 10MB)
    if (file.size > 10 * 1024 * 1024) {
        showError('Файл занадто великий (максимум 10MB)');
        return;
    }
    
    try {
        // Показати індикатор завантаження
        const loadingMessage = addLoadingMessage();
        
        // Завантаження в Firebase Storage
        const fileName = `${type}s/${Date.now()}_${file.name}`;
        const fileRef = storageRef(storage, fileName);
        
        const snapshot = await uploadBytes(fileRef, file);
        const downloadURL = await getDownloadURL(snapshot.ref);
        
        // Видалити індикатор завантаження
        removeLoadingMessage(loadingMessage);
        
        // Відправити повідомлення з медіа
        const messageData = {
            text: '',
            sender: currentUser,
            timestamp: serverTimestamp(),
            type: 'user',
            media: {
                type: type,
                url: downloadURL,
                name: file.name,
                size: file.size
            }
        };
        
        await push(ref(db, 'messages'), messageData);
        
    } catch (error) {
        console.error('Помилка завантаження файлу:', error);
        showError('Не вдалося завантажити файл');
    }
    
    // Очистити input
    event.target.value = '';
    attachmentMenu.style.display = 'none';
}

// Рендеринг повідомлень
function renderMessages() {
    if (!messagesArea) return;
    
    messagesArea.innerHTML = '';
    
    messages.forEach((message, index) => {
        const messageElement = createMessageElement(message, index);
        messagesArea.appendChild(messageElement);
    });
    
    scrollToBottom();
}

// Створення елемента повідомлення
function createMessageElement(message, index) {
    const messageDiv = document.createElement('div');
    messageDiv.className = `message ${message.type}`;
    messageDiv.dataset.messageId = message.id;
    
    // Аватар для інших користувачів
    if (message.type !== 'user') {
        const character = characters[message.character] || {};
        const avatarDiv = document.createElement('div');
        avatarDiv.className = 'message-avatar';
        avatarDiv.style.backgroundColor = character.color || '#2b5278';
        
        if (character.avatar && character.avatar.startsWith('http')) {
            avatarDiv.innerHTML = `<img src="${character.avatar}" alt="${character.name}">`;
        } else {
            avatarDiv.textContent = character.avatar || '👻';
        }
        
        messageDiv.appendChild(avatarDiv);
    }
    
    // Контент повідомлення
    const contentDiv = document.createElement('div');
    contentDiv.className = 'message-content';
    
    // Відповідь на повідомлення
    if (message.replyTo) {
        const replyDiv = createReplyElement(message.replyTo);
        contentDiv.appendChild(replyDiv);
    }
    
    // Ім'я відправника (для інших користувачів)
    if (message.type !== 'user') {
        const character = characters[message.character] || {};
        const senderDiv = document.createElement('div');
        senderDiv.className = 'message-sender';
        senderDiv.textContent = character.name || 'Невідомий';
        contentDiv.appendChild(senderDiv);
    }
    
    // Медіа контент
    if (message.media) {
        const mediaDiv = createMediaElement(message.media);
        contentDiv.appendChild(mediaDiv);
    }
    
    // Текст повідомлення
    if (message.text) {
        const textDiv = document.createElement('div');
        textDiv.className = 'message-text';
        textDiv.textContent = message.text;
        contentDiv.appendChild(textDiv);
    }
    
    // Час відправки
    const timeDiv = document.createElement('div');
    timeDiv.className = 'message-time';
    timeDiv.textContent = formatTime(message.timestamp);
    contentDiv.appendChild(timeDiv);
    
    messageDiv.appendChild(contentDiv);
    
    // Додати свайп для відповіді
    if (message.type !== 'user') {
        addSwipeToReply(messageDiv, message);
    }
    
    return messageDiv;
}

// Створення елемента відповіді
function createReplyElement(replyToId) {
    const replyMessage = messages.find(m => m.id === replyToId);
    if (!replyMessage) return document.createElement('div');
    
    const replyDiv = document.createElement('div');
    replyDiv.className = 'reply-to';
    
    const senderDiv = document.createElement('div');
    senderDiv.className = 'reply-sender';
    
    if (replyMessage.type === 'user') {
        senderDiv.textContent = 'Ви';
    } else {
        const character = characters[replyMessage.character] || {};
        senderDiv.textContent = character.name || 'Невідомий';
    }
    
    const textDiv = document.createElement('div');
    textDiv.className = 'reply-text';
    textDiv.textContent = replyMessage.text || (replyMessage.media ? `${replyMessage.media.type === 'photo' ? '📷' : '🎥'} ${replyMessage.media.name}` : '');
    
    replyDiv.appendChild(senderDiv);
    replyDiv.appendChild(textDiv);
    
    return replyDiv;
}

// Створення медіа елемента
function createMediaElement(media) {
    const mediaDiv = document.createElement('div');
    mediaDiv.className = 'message-media';
    
    if (media.type === 'photo') {
        const img = document.createElement('img');
        img.src = media.url;
        img.alt = media.name;
        img.addEventListener('click', () => openMediaViewer(media.url, 'image'));
        mediaDiv.appendChild(img);
    } else if (media.type === 'video') {
        const video = document.createElement('video');
        video.src = media.url;
        video.controls = true;
        video.preload = 'metadata';
        mediaDiv.appendChild(video);
    }
    
    return mediaDiv;
}

// Додавання свайпу для відповіді
function addSwipeToReply(messageElement, message) {
    const replyBtn = document.createElement('div');
    replyBtn.className = 'swipe-reply';
    replyBtn.innerHTML = '↶';
    replyBtn.addEventListener('click', () => replyToMessage(message));
    messageElement.appendChild(replyBtn);
}

// Touch події для свайпів
function handleTouchStart(e) {
    touchStartX = e.touches[0].clientX;
    touchStartY = e.touches[0].clientY;
}

function handleTouchMove(e) {
    if (!touchStartX || !touchStartY) return;
    
    const touchX = e.touches[0].clientX;
    const touchY = e.touches[0].clientY;
    
    const diffX = touchStartX - touchX;
    const diffY = touchStartY - touchY;
    
    // Перевірка на горизонтальний свайп
    if (Math.abs(diffX) > Math.abs(diffY) && Math.abs(diffX) > swipeThreshold) {
        const messageElement = e.target.closest('.message');
        if (messageElement && !messageElement.classList.contains('user')) {
            messageElement.classList.add('swiping');
        }
    }
}

function handleTouchEnd(e) {
    const messageElement = e.target.closest('.message');
    if (messageElement && messageElement.classList.contains('swiping')) {
        messageElement.classList.remove('swiping');
        
        // Якщо свайп був достатньо довгим, почати відповідь
        const messageId = messageElement.dataset.messageId;
        const message = messages.find(m => m.id === messageId);
        if (message) {
            replyToMessage(message);
        }
    }
    
    touchStartX = 0;
    touchStartY = 0;
}

// Відповідь на повідомлення
function replyToMessage(message) {
    replyingTo = message.id;
    
    // Показати превью відповіді
    if (message.type === 'user') {
        replySender.textContent = 'Ви';
    } else {
        const character = characters[message.character] || {};
        replySender.textContent = character.name || 'Невідомий';
    }
    
    replyText.textContent = message.text || (message.media ? `${message.media.type === 'photo' ? '📷' : '🎥'} ${message.media.name}` : '');
    replyPreview.style.display = 'block';
    
    messageInput.focus();
}

// Скасування відповіді
function cancelReply() {
    replyingTo = null;
    replyPreview.style.display = 'none';
}

// Перемикання меню вкладень
function toggleAttachmentMenu() {
    const isVisible = attachmentMenu.style.display === 'flex';
    attachmentMenu.style.display = isVisible ? 'none' : 'flex';
}

// Оновлення видимості кнопки відправки
function updateSendButtonVisibility() {
    const hasText = messageInput.value.trim().length > 0;
    sendButton.classList.toggle('visible', hasText);
}

// Показати індикатор набору тексту
function showTypingIndicator(show) {
    if (typingIndicator) {
        typingIndicator.style.display = show ? 'block' : 'none';
        if (show) {
            scrollToBottom();
        }
    }
}

// Прокрутка вниз
function scrollToBottom() {
    if (messagesArea) {
        setTimeout(() => {
            messagesArea.scrollTop = messagesArea.scrollHeight;
        }, 100);
    }
}

// Додавання повідомлення завантаження
function addLoadingMessage() {
    const loadingDiv = document.createElement('div');
    loadingDiv.className = 'message user loading';
    loadingDiv.innerHTML = `
        <div class="message-content">
            <div class="message-text">Завантаження...</div>
        </div>
    `;
    messagesArea.appendChild(loadingDiv);
    scrollToBottom();
    return loadingDiv;
}

// Видалення повідомлення завантаження
function removeLoadingMessage(loadingElement) {
    if (loadingElement && loadingElement.parentNode) {
        loadingElement.parentNode.removeChild(loadingElement);
    }
}

// Застосування налаштувань чату
function applyChatSettings() {
    if (chatSettings.title) {
        chatTitle.textContent = chatSettings.title;
        document.title = chatSettings.title;
    }
    
    if (chatSettings.avatar) {
        if (chatSettings.avatar.startsWith('http')) {
            chatAvatar.innerHTML = `<img src="${chatSettings.avatar}" alt="Chat Avatar">`;
        } else {
            chatAvatar.textContent = chatSettings.avatar;
        }
    }
    
    if (chatSettings.background) {
        if (chatSettings.background.startsWith('http')) {
            document.body.style.setProperty('--bg-image', `url(${chatSettings.background})`);
        } else {
            // Застосування градієнтів
            const gradients = {
                gradient1: 'linear-gradient(135deg, #0f1419 0%, #17212b 100%)',
                gradient2: 'linear-gradient(135deg, #1a0f0f 0%, #2d1717 100%)',
                gradient3: 'linear-gradient(135deg, #1a0f1a 0%, #2d172d 100%)',
                gradient4: 'linear-gradient(135deg, #0f1a0f 0%, #172d17 100%)',
                gradient5: 'linear-gradient(135deg, #0f1a1a 0%, #17272d 100%)'
            };
            
            if (gradients[chatSettings.background]) {
                document.body.style.background = gradients[chatSettings.background];
            }
        }
    }
    
    if (chatSettings.colorScheme && chatSettings.colorScheme !== 'default') {
        applyColorScheme(chatSettings.colorScheme);
    }
}

// Застосування кольорової схеми
function applyColorScheme(scheme) {
    const schemes = {
        dark: {
            '--bg-color': '#000000',
            '--header-bg': '#1c1c1e',
            '--message-bg': '#2c2c2e',
            '--accent-color': '#007aff'
        },
        blue: {
            '--bg-color': '#001122',
            '--header-bg': '#002244',
            '--message-bg': '#003366',
            '--accent-color': '#0088ff'
        },
        green: {
            '--bg-color': '#001100',
            '--header-bg': '#002200',
            '--message-bg': '#003300',
            '--accent-color': '#00ff88'
        },
        red: {
            '--bg-color': '#110000',
            '--header-bg': '#220000',
            '--message-bg': '#330000',
            '--accent-color': '#ff0088'
        },
        purple: {
            '--bg-color': '#110011',
            '--header-bg': '#220022',
            '--message-bg': '#330033',
            '--accent-color': '#8800ff'
        }
    };
    
    if (schemes[scheme]) {
        const root = document.documentElement;
        Object.entries(schemes[scheme]).forEach(([property, value]) => {
            root.style.setProperty(property, value);
        });
    }
}

// Форматування часу
function formatTime(timestamp) {
    if (!timestamp) return '';
    
    const date = new Date(timestamp);
    const now = new Date();
    
    if (date.toDateString() === now.toDateString()) {
        return date.toLocaleTimeString('uk-UA', { 
            hour: '2-digit', 
            minute: '2-digit' 
        });
    } else {
        return date.toLocaleDateString('uk-UA', { 
            day: '2-digit', 
            month: '2-digit',
            hour: '2-digit', 
            minute: '2-digit' 
        });
    }
}

// Відкриття переглядача медіа
function openMediaViewer(url, type) {
    const viewer = document.createElement('div');
    viewer.style.cssText = `
        position: fixed;
        top: 0;
        left: 0;
        right: 0;
        bottom: 0;
        background: rgba(0,0,0,0.9);
        display: flex;
        align-items: center;
        justify-content: center;
        z-index: 10000;
        cursor: pointer;
    `;
    
    if (type === 'image') {
        const img = document.createElement('img');
        img.src = url;
        img.style.cssText = `
            max-width: 90%;
            max-height: 90%;
            object-fit: contain;
        `;
        viewer.appendChild(img);
    }
    
    viewer.addEventListener('click', () => {
        document.body.removeChild(viewer);
    });
    
    document.body.appendChild(viewer);
}

// Показати помилку
function showError(message) {
    const errorDiv = document.createElement('div');
    errorDiv.style.cssText = `
        position: fixed;
        top: 20px;
        left: 50%;
        transform: translateX(-50%);
        background: #dc3545;
        color: white;
        padding: 12px 24px;
        border-radius: 8px;
        z-index: 10000;
        font-size: 14px;
    `;
    errorDiv.textContent = message;
    
    document.body.appendChild(errorDiv);
    
    setTimeout(() => {
        if (document.body.contains(errorDiv)) {
            document.body.removeChild(errorDiv);
        }
    }, 3000);
}

// Експорт функцій для глобального використання
window.cancelReply = cancelReply;
window.toggleAttachmentMenu = toggleAttachmentMenu;