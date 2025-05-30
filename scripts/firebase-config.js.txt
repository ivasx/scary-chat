import { initializeApp } from "https://www.gstatic.com/firebasejs/10.12.0/firebase-app.js";
import { getDatabase } from "https://www.gstatic.com/firebasejs/10.12.0/firebase-database.js";
import { getStorage } from "https://www.gstatic.com/firebasejs/10.12.0/firebase-storage.js";

const firebaseConfig = {
  apiKey: "AIzaSyBEEfn7xzGqKa1Cq9KOeeDXv836wMreFrA",
  authDomain: "scary-72f98.firebaseapp.com",
  databaseURL: "https://scary-72f98-default-rtdb.firebaseio.com",
  projectId: "scary-72f98",
  storageBucket: "scary-72f98.appspot.com",
  messagingSenderId: "450754639280",
  appId: "1:450754639280:web:5550f2cfeb4aac1f0e1275",
  measurementId: "G-6FKRD4WWHG"
};

const app = initializeApp(firebaseConfig);
export const db = getDatabase(app);
export const storage = getStorage(app);