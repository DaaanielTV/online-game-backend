Backend of https://github.com/DaaanielTV/online-game-frontend


Basierend auf dem Codebase werde ich eine REST API und WebSocket-Integration für das Backend vorschlagen. Hier ist die Implementierung:

Zuerst erweitern wir den Express Server um API Routes:
Dann erstellen wir einen API Client für das Frontend:
Integration in die Game Engine:
Aktualisieren der package.json:
Diese Implementierung bietet:

REST API für spielstandunabhängige Operationen
WebSocket für Echtzeit-Spielkommunikation
Client-seitige API-Wrapper-Klasse
Integration in die Game Engine
Behandlung von Verbindungen/Trennungen
Raum-Management für Multiplayer
Position-Synchronisation zwischen Spielern
Um es zu verwenden:

Installiere die neuen Abhängigkeiten: npm install
Starte den Server: npm start
Im Frontend wird die GameAPI-Klasse automatisch von der Game Engine verwendet
Die Kommunikation läuft über:

HTTP/REST für spielstandunabhängige Aktionen
WebSocket für Echtzeit-Gameplay
JSON für Datenaustausch

const express = require('express');
const path = require('path');
const WebSocket = require('ws');
const cors = require('cors');

const app = express();
const server = require('http').createServer(app);
const wss = new WebSocket.Server({ server });

// Middleware
app.use(cors());
app.use(express.json());
app.use(express.static(__dirname));

// Game state
const gameState = {
    players: new Map(),
    rooms: new Map()
};

// REST API Routes
app.get('/api/games', (req, res) => {
    res.json(Array.from(gameState.rooms.values()).map(room => ({
        id: room.id,
        players: room.players.size,
        state: room.state
    })));
});

app.post('/api/games', (req, res) => {
    const roomId = Date.now().toString();
    const room = new Room(roomId);
    gameState.rooms.set(roomId, room);
    res.json({ roomId });
});

// WebSocket connection handling
wss.on('connection', (ws) => {
    const playerId = Date.now().toString();
    
    gameState.players.set(playerId, {
        ws,
        room: null,
        position: { x: 0, y: 0 }
    });

    // Send initial state
    ws.send(JSON.stringify({
        type: 'init',
        playerId,
        rooms: Array.from(gameState.rooms.values()).map(room => ({
            id: room.id,
            players: room.players.size
        }))
    }));

    // Handle messages
    ws.on('message', (message) => {
        const data = JSON.parse(message);
        handleMessage(playerId, data);
    });

    // Handle disconnection
    ws.on('close', () => handleDisconnect(playerId));
});

// Message handler
function handleMessage(playerId, data) {
    const player = gameState.players.get(playerId);
    if (!player) return;

    switch(data.type) {
        case 'join_room':
            handleJoinRoom(playerId, data.roomId);
            break;
        case 'position':
            handlePosition(playerId, data.x, data.y);
            break;
        // Add more message handlers as needed
    }
}

// Start server
const PORT = process.env.PORT || 3000;
server.listen(PORT, () => {
    console.log(`Server running at http://localhost:${PORT}`);
    console.log('WebSocket server active');
    console.log('Press Ctrl+C to stop');
});

Dann erstellen wir einen API Client für das Frontend:

class GameAPI {
    constructor(baseURL = 'http://localhost:3000') {
        this.baseURL = baseURL;
        this.ws = null;
        this.messageHandlers = new Map();
    }

    // REST Methods
    async getGames() {
        const response = await fetch(`${this.baseURL}/api/games`);
        return response.json();
    }

    async createGame() {
        const response = await fetch(`${this.baseURL}/api/games`, {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json'
            }
        });
        return response.json();
    }

    // WebSocket Methods
    connect() {
        this.ws = new WebSocket(`ws://${this.baseURL.replace('http://', '')}`);
        
        this.ws.onmessage = (event) => {
            const data = JSON.parse(event.data);
            const handler = this.messageHandlers.get(data.type);
            if (handler) {
                handler(data);
            }
        };

        return new Promise((resolve, reject) => {
            this.ws.onopen = () => resolve();
            this.ws.onerror = (error) => reject(error);
        });
    }

    onMessage(type, handler) {
        this.messageHandlers.set(type, handler);
    }

    sendMessage(type, data) {
        if (this.ws && this.ws.readyState === WebSocket.OPEN) {
            this.ws.send(JSON.stringify({
                type,
                ...data
            }));
        }
    }

    disconnect() {
        if (this.ws) {
            this.ws.close();
            this.ws = null;
        }
    }
}

export default GameAPI;

Integration in die Game Engine:

import GameAPI from '../../js/api-client.js';

class GameEngine {
    constructor(canvasId) {
        // ... existing constructor code ...
        
        this.api = new GameAPI();
        this.setupNetworking();
    }

    async setupNetworking() {
        try {
            // Connect to server
            await this.api.connect();

            // Handle network messages
            this.api.onMessage('init', (data) => {
                this.playerId = data.playerId;
                this.availableRooms = data.rooms;
            });

            this.api.onMessage('player_joined', (data) => {
                // Handle new player joining
                this.addNetworkPlayer(data);
            });

            this.api.onMessage('position', (data) => {
                // Update other player positions
                this.updatePlayerPosition(data);
            });

        } catch (error) {
            console.error('Failed to connect to game server:', error);
        }
    }

    updatePlayerPosition(position) {
        // Send position update to server
        this.api.sendMessage('position', {
            x: position.x,
            y: position.y
        });
    }
}

Aktualisieren der package.json:

{
  "name": "online-game",
  "version": "1.0.0",
  "description": "Online game collection",
  "main": "server.js",
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "ws": "^8.2.3",
    "cors": "^2.8.5"
  },
  "devDependencies": {
    "nodemon": "^2.0.15"
  }
}

