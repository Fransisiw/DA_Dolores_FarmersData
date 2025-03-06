// Backend: Node.js with Express
const express = require('express');
const sqlite3 = require('sqlite3').verbose();
const cors = require('cors');
const path = require('path');
const { promisify } = require('util');

const app = express();
app.use(cors());
app.use(express.json());
app.use(express.static(path.join(__dirname, 'public')));

// Database connection
const db = new sqlite3.Database('./farmers.db', (err) => {
    if (err) {
        console.error('Database connection failed:', err);
    } else {
        console.log('Connected to SQLite database');
        // Ensure tables exist
        createTablesIfNotExist();
    }
});

// Helper to run queries as promises
const dbRun = promisify(db.run.bind(db));
const dbAll = promisify(db.all.bind(db));
const dbGet = promisify(db.get.bind(db));

// Create tables if they don't exist
function createTablesIfNotExist() {
    // SQLite doesn't support AUTO_INCREMENT or TIMESTAMP keywords
    const createFoldersTable = `
        CREATE TABLE IF NOT EXISTS folders (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT NOT NULL,
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP
        )
    `;

    const createItemsTable = `
        CREATE TABLE IF NOT EXISTS items (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            folder_id INTEGER,
            name TEXT NOT NULL,
            description TEXT,
            contact_info TEXT,
            location TEXT,
            is_active INTEGER DEFAULT 1,
            is_verified INTEGER DEFAULT 0,
            notes TEXT,
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
            FOREIGN KEY (folder_id) REFERENCES folders(id) ON DELETE CASCADE
        )
    `;

    db.serialize(() => {
        db.run(createFoldersTable, (err) => {
            if (err) {
                console.error('Error creating folders table:', err);
                return;
            }
            console.log('Folders table created or already exists');
        });

        db.run(createItemsTable, (err) => {
            if (err) {
                console.error('Error creating items table:', err);
                return;
            }
            console.log('Items table created or already exists');
        });
    });
}

// API Routes

// Folders
app.get('/api/folders', (req, res) => {
    db.all('SELECT * FROM folders ORDER BY created_at DESC', (err, results) => {
        if (err) return res.status(500).json({ error: err.message });
        res.json(results);
    });
});

app.post('/api/folders', (req, res) => {
    const { name } = req.body;
    if (!name) return res.status(400).json({ error: 'Name is required' });

    db.run('INSERT INTO folders (name) VALUES (?)', [name], function(err) {
        if (err) return res.status(500).json({ error: err.message });
        res.json({ id: this.lastID, name });
    });
});

app.put('/api/folders/:id', (req, res) => {
    const { name } = req.body;
    if (!name) return res.status(400).json({ error: 'Name is required' });

    db.run('UPDATE folders SET name = ? WHERE id = ?', [name, req.params.id], function(err) {
        if (err) return res.status(500).json({ error: err.message });
        res.json({ id: parseInt(req.params.id), name });
    });
});

app.delete('/api/folders/:id', (req, res) => {
    db.run('DELETE FROM folders WHERE id = ?', [req.params.id], function(err) {
        if (err) return res.status(500).json({ error: err.message });
        res.json({ message: 'Folder deleted' });
    });
});

// Items
app.get('/api/folders/:folderId/items', (req, res) => {
    db.all('SELECT * FROM items WHERE folder_id = ? ORDER BY created_at DESC', [req.params.folderId], (err, results) => {
        if (err) return res.status(500).json({ error: err.message });
        res.json(results);
    });
});

app.post('/api/items', (req, res) => {
    const { folder_id, name, description, contact_info, location, is_active, is_verified, notes } = req.body;
    if (!name || !folder_id) return res.status(400).json({ error: 'Name and folder_id are required' });

    const is_active_value = is_active !== undefined ? (is_active ? 1 : 0) : 1;
    const is_verified_value = is_verified !== undefined ? (is_verified ? 1 : 0) : 0;

    db.run(`INSERT INTO items 
            (folder_id, name, description, contact_info, location, is_active, is_verified, notes) 
            VALUES (?, ?, ?, ?, ?, ?, ?, ?)`, 
        [folder_id, name, description || '', contact_info || '', location || '', 
         is_active_value, is_verified_value, notes || ''], function(err) {
        if (err) return res.status(500).json({ error: err.message });
        res.json({ 
            id: this.lastID, 
            folder_id, 
            name, 
            description,
            contact_info,
            location,
            is_active: !!is_active_value,
            is_verified: !!is_verified_value,
            notes
        });
    });
});

app.put('/api/items/:id', (req, res) => {
    const { name, description, contact_info, location, is_active, is_verified, notes } = req.body;
    if (!name) return res.status(400).json({ error: 'Name is required' });

    const is_active_value = is_active !== undefined ? (is_active ? 1 : 0) : 1;
    const is_verified_value = is_verified !== undefined ? (is_verified ? 1 : 0) : 0;

    db.run(`UPDATE items SET 
            name = ?, 
            description = ?, 
            contact_info = ?,
            location = ?,
            is_active = ?,
            is_verified = ?,
            notes = ?
            WHERE id = ?`, 
        [name, description || '', contact_info || '', location || '', 
         is_active_value, is_verified_value, notes || '', req.params.id], function(err) {
        if (err) return res.status(500).json({ error: err.message });
        res.json({ 
            id: parseInt(req.params.id), 
            name, 
            description,
            contact_info,
            location,
            is_active: !!is_active_value,
            is_verified: !!is_verified_value,
            notes
        });
    });
});

// Search endpoint
app.get('/api/search', (req, res) => {
    const { query } = req.query;
    if (!query) return res.status(400).json({ error: 'Search query is required' });

    const searchQuery = `%${query}%`;
    const sql = `
        SELECT i.*, f.name as folder_name
        FROM items i 
        JOIN folders f ON i.folder_id = f.id
        WHERE i.name LIKE ? 
        OR i.description LIKE ? 
        OR i.contact_info LIKE ? 
        OR i.location LIKE ?
        OR i.notes LIKE ?
        ORDER BY i.created_at DESC
    `;

    db.all(sql, [searchQuery, searchQuery, searchQuery, searchQuery, searchQuery], (err, results) => {
        if (err) return res.status(500).json({ error: err.message });
        res.json(results);
    });
});

app.delete('/api/items/:id', (req, res) => {
    db.run('DELETE FROM items WHERE id = ?', [req.params.id], function(err) {
        if (err) return res.status(500).json({ error: err.message });
        res.json({ message: 'Item deleted' });
    });
});

// Serve the HTML file for any other route
app.get('*', (req, res) => {
  res.sendFile(path.join(__dirname, 'public', 'index.html'));
});

// Start server
const PORT = process.env.PORT || 5000;
app.listen(PORT, '0.0.0.0', () => {
    console.log(`Server running on port ${PORT}`);
});
