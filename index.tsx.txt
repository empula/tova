/**
 * @license
 * SPDX-License-Identifier: Apache-2.0
 */
import { GoogleGenAI, Chat, GenerateContentResponse } from "@google/genai";

const API_KEY = process.env.API_KEY;

const chatContainer = document.getElementById('chat-container') as HTMLDivElement | null;
const messageInput = document.getElementById('message-input') as HTMLInputElement | null;
const sendButton = document.getElementById('send-button') as HTMLButtonElement | null;
const appWrapper = document.getElementById('app-wrapper') as HTMLDivElement | null;
const marketDataArea = document.getElementById('market-data-area') as HTMLDivElement | null;
const clockDisplay = document.getElementById('clock-display') as HTMLDivElement | null;

function displayErrorToUser(message: string, isCritical: boolean = false, targetElement?: HTMLElement | null) {
    const displayTarget = targetElement || chatContainer;

    if (displayTarget && displayTarget === chatContainer) {
        const errorElement = document.createElement('div');
        errorElement.classList.add('message', 'error-message');
        errorElement.textContent = `Hata: ${message}`;
        displayTarget.appendChild(errorElement);
        displayTarget.scrollTop = displayTarget.scrollHeight;
    } else if (appWrapper) {
        const errorDiv = document.createElement('div');
        errorDiv.textContent = isCritical ? `Kritik Hata: ${message}` : `Hata: ${message}`;
        errorDiv.style.color = '#ffabab';
        errorDiv.style.backgroundColor = '#5c2323';
        errorDiv.style.padding = '10px';
        errorDiv.style.textAlign = 'center';
        errorDiv.style.margin = '10px auto';
        errorDiv.style.borderRadius = '8px';
        errorDiv.style.maxWidth = '90%';
        const header = document.getElementById('app-header');
        if (header && header.parentNode === appWrapper) {
            header.after(errorDiv);
        } else {
            appWrapper.prepend(errorDiv);
        }
    } else {
        alert(isCritical ? `Kritik Hata: ${message}` : `Hata: ${message}`);
        console.error(isCritical ? `Kritik Hata: ${message}` : `Error: ${message}`);
    }
     if (isCritical && sendButton) sendButton.disabled = true;
     if (isCritical && messageInput) messageInput.disabled = true;
}


if (!API_KEY) {
    displayErrorToUser('Gemini API_KEY ortam değişkeni ayarlanmamış. Sohbet işlevselliği etkilenecektir.', true);
}

const ai = new GoogleGenAI({ apiKey: API_KEY! });
let chat: Chat;


function displayMessage(text: string, sender: 'user' | 'model', isStreaming: boolean = false): HTMLDivElement | null {
    if (!chatContainer) {
        console.error("Sohbet konteyneri bulunamadı. Mesaj gösterilemiyor.");
        return null;
    }
    const messageElement = document.createElement('div');
    messageElement.classList.add('message', `${sender}-message`);
    messageElement.textContent = text;

    if (sender === 'model' && isStreaming) {
        messageElement.classList.add('streaming');
    }
    chatContainer.appendChild(messageElement);
    chatContainer.scrollTop = chatContainer.scrollHeight;
    return messageElement;
}

async function initializeChat() {
    if (!API_KEY) {
        console.warn("API_KEY ayarlanmadığı için sohbet başlatma atlanıyor.");
        return;
    }
    try {
        chat = ai.chats.create({
            model: 'gemini-2.5-flash-preview-04-17',
            config: { systemInstruction: "Sen TOVA adında, Türkçe konuşan ve yardımsever bir yapay zeka asistanısın." }
        });
        displayMessage("Hoşgeldiniz! Size nasıl yardımcı olabilirim?", 'model');
    } catch (error) {
        console.error("Sohbet başlatma başarısız:", error);
        const errorMessage = error instanceof Error ? error.message : String(error);
        displayErrorToUser(`Sohbet başlatılamadı. ${errorMessage}`, true);
    }
}

async function sendMessage() {
    if (!API_KEY || !chat) {
        displayErrorToUser("Sohbet başlatılmamış (API Anahtarı eksik veya kurulum başarısız olmuş olabilir).");
        return;
    }
    if (!messageInput || !sendButton) {
        displayErrorToUser("Giriş elemanları bulunamadı.");
        return;
    }

    const messageText = messageInput.value.trim();
    if (!messageText) return;

    displayMessage(messageText, 'user');
    messageInput.value = '';
    sendButton.disabled = true;
    messageInput.disabled = true;

    let modelMessageElement: HTMLDivElement | null = null;

    try {
        const stream = await chat.sendMessageStream({ message: messageText });
        let firstChunk = true;
        let fullResponseText = "";

        for await (const chunk of stream) {
            const chunkText = chunk.text;
            fullResponseText += chunkText;

            if (firstChunk) {
                modelMessageElement = displayMessage(chunkText, 'model', true);
                firstChunk = false;
            } else if (modelMessageElement) {
                modelMessageElement.textContent = fullResponseText;
            }
            if (chatContainer) {
                 chatContainer.scrollTop = chatContainer.scrollHeight;
            }
        }
        if (modelMessageElement) {
            modelMessageElement.classList.remove('streaming');
        }
    } catch (error) {
        console.error("Mesaj gönderilirken hata:", error);
        const errorMessage = error instanceof Error ? error.message : String(error);
        displayErrorToUser(`Yanıt alınamadı. ${errorMessage}`);
        if (modelMessageElement) {
            modelMessageElement.classList.add('error-message'); 
            modelMessageElement.textContent = "TOVA'dan yanıt alınırken hata oluştu.";
        } else {
            displayMessage("TOVA'dan yanıt alınırken hata oluştu.", 'model');
        }
    } finally {
        if (sendButton) sendButton.disabled = false;
        if (messageInput) {
            messageInput.disabled = false;
            messageInput.focus();
        }
    }
}

const marketPairs = [
    { id: 'bitcoin', name: 'BTC/USD', vs_currency: 'usd' },
    { id: 'ethereum', name: 'ETH/USD', vs_currency: 'usd' }
];

function updateMarketDataItem(pairName: string, value: string | number, isError: boolean = false) {
    if (!marketDataArea) return;
    let itemElement = document.getElementById(`market-item-${pairName.replace('/', '-')}`);
    if (!itemElement) {
        itemElement = document.createElement('div');
        itemElement.id = `market-item-${pairName.replace('/', '-')}`;
        itemElement.classList.add('market-item');
        
        const nameSpan = document.createElement('span');
        nameSpan.classList.add('pair-name');
        nameSpan.textContent = `${pairName}: `;
        
        const valueSpan = document.createElement('span');
        valueSpan.classList.add('pair-value');
        
        itemElement.appendChild(nameSpan);
        itemElement.appendChild(valueSpan);
        marketDataArea.appendChild(itemElement);
    }
    
    const valueSpan = itemElement.querySelector('.pair-value') as HTMLSpanElement;
    if (valueSpan) {
        if (isError) {
            valueSpan.textContent = typeof value === 'string' ? value : 'Hata';
            valueSpan.classList.add('error');
        } else {
            valueSpan.textContent = typeof value === 'number' 
                ? `$${value.toLocaleString('en-US', { minimumFractionDigits: 2, maximumFractionDigits: 2 })}` 
                : String(value);
            valueSpan.classList.remove('error');
        }
    }
}


async function fetchMarketValue() {
    if (!marketDataArea) {
        console.warn("Piyasa verisi alanı bulunamadı.");
        return;
    }
    
    // Remove elements for pairs that are no longer in marketPairs
    if (marketDataArea) {
        const currentPairIdsInDom = Array.from(marketDataArea.children).map(child => child.id.replace('market-item-', '').replace('-', '/'));
        const activePairNames = marketPairs.map(p => p.name);
        currentPairIdsInDom.forEach(domPairId => {
            if (!activePairNames.includes(domPairId)) {
                const elementToRemove = document.getElementById(`market-item-${domPairId.replace('/', '-')}`);
                if (elementToRemove) {
                    marketDataArea.removeChild(elementToRemove);
                }
            }
        });
    }


    const idsToFetch = marketPairs.map(p => p.id).join(',');
    const vsCurrency = 'usd'; // Assuming all are vs USD for simplicity with CoinGecko

    try {
        if (idsToFetch) { // Only fetch if there are pairs to fetch
            const response = await fetch(`https://api.coingecko.com/api/v3/simple/price?ids=${idsToFetch}&vs_currencies=${vsCurrency}`);
            if (!response.ok) {
                throw new Error(`CoinGecko API isteği başarısız: ${response.status} ${response.statusText}`);
            }
            const data = await response.json();

            marketPairs.forEach(pair => {
                if (data[pair.id] && data[pair.id][vsCurrency]) {
                    updateMarketDataItem(pair.name, data[pair.id][vsCurrency]);
                } else {
                    updateMarketDataItem(pair.name, "Veri Yok", true);
                    console.warn(`CoinGecko'dan ${pair.name} için veri alınamadı.`);
                }
            });
        } else { // If no pairs are defined, ensure existing DOM elements are cleared or updated
             marketPairs.forEach(pair => { // This might be redundant if clearing logic above is robust
                updateMarketDataItem(pair.name, "Veri Yok", true);
            });
        }
    } catch (error) {
        console.error("Piyasa değeri alınamadı:", error);
        marketPairs.forEach(pair => { // Update all defined pairs with an error message
            updateMarketDataItem(pair.name, "Yüklenemedi", true);
        });
    }
}

function updateClock() {
    if (!clockDisplay) {
        return;
    }
    const now = new Date();
    const hours = String(now.getHours()).padStart(2, '0');
    const minutes = String(now.getMinutes()).padStart(2, '0');
    const seconds = String(now.getSeconds()).padStart(2, '0');
    clockDisplay.textContent = `${hours}:${minutes}:${seconds}`;
}

// Initial setup
if (appWrapper && chatContainer && messageInput && sendButton && marketDataArea && clockDisplay) {
    sendButton.addEventListener('click', sendMessage);
    messageInput.addEventListener('keypress', (event) => {
        if (event.key === 'Enter' && !event.shiftKey) {
            event.preventDefault();
            sendMessage();
        }
    });
    
    initializeChat();
    fetchMarketValue(); 
    setInterval(fetchMarketValue, 60000); 

    updateClock(); 
    setInterval(updateClock, 1000); 

} else {
    let missingElements = [];
    if (!appWrapper) missingElements.push("app-wrapper");
    if (!chatContainer) missingElements.push("chat-container (sohbet için)");
    if (!messageInput) missingElements.push("message-input (sohbet için)");
    if (!sendButton) missingElements.push("send-button (sohbet için)");
    if (!marketDataArea) missingElements.push("market-data-area (piyasa verileri için)");
    if (!clockDisplay) missingElements.push("clock-display (saat için)");
    
    const errorMessageText = `Gerekli HTML elemanları bulunamadı: ${missingElements.join(', ')}. Bazı işlevler çalışmayabilir.`;
    console.error(errorMessageText);
    // Attempt to display error in body if appWrapper is missing, otherwise use appWrapper if chatContainer is not the target.
    displayErrorToUser(errorMessageText, true, appWrapper || document.body); 
}
