<script setup lang="ts">
import { ref, onMounted, watch } from 'vue';
import PouchDB from 'pouchdb';
// Import du plugin de recherche (Obligatoire pour .find() et .createIndex())
import PouchDBFind from 'pouchdb-find';

// Activation du plugin de recherche dans PouchDB
PouchDB.plugin(PouchDBFind);

// ================================================================
// I. INTERFACES
// ================================================================

interface Post {
    _id: string;
    _rev?: string;
    type: 'Post'; // Clé de partitionnement/filtrage
    title: string;
    content: string;
    likes: number;
    author: string;
    creationDate: string;
}

// ================================================================
// II. FACTORY (Génération de données)
// ================================================================

const postFactory = (index: number): Post => {
    const timestamp = new Date().toISOString();
    return {
        _id: `post_${new Date().getTime()}_${index}`, // ID unique basé sur le temps
        type: 'Post',
        title: `Message #${index}`,
        content: `Ceci est le contenu du message numéro ${index}. C'est une information importante pour votre projet.`,
        likes: Math.floor(Math.random() * 100), // Nombre de likes aléatoire entre 0 et 100
        author: 'utilisateur_demo',
        creationDate: timestamp
    };
}

// ================================================================
// III. LOGIQUE DATABASE
// ================================================================

// !!! ATTENTION : Assurez-vous que l'URL et le nom de la base de données correspondent à votre CouchDB !!!
const remoteDbUrl = 'http://admin:admin@localhost:5984/infra_53_0850';
const LOCAL_DB_NAME = 'local_infra_53_0850';

let remoteDb: PouchDB.Database | null = null;
let localDb: PouchDB.Database | null = null;

// Variables réactives (État de l'interface)
const documents = ref<Post[]>([]); 
const isOnline = ref(true); 
const searchQuery = ref(''); // Le texte tapé dans la recherche
const searchStatus = ref(''); // Message d'info (ex: "3 résultats trouvés")

let syncHandler: PouchDB.Replication.Sync<any> | null = null;

// --- 1. Initialisation & Création d'Index ---
const initDatabase = async () => {
    try {
        remoteDb = new PouchDB(remoteDbUrl);
        localDb = new PouchDB(LOCAL_DB_NAME);
        console.log('✅ Bases de données locale et distante connectées.');

        // CRUCIAL : Création des INDEX (obligatoire pour .find() et la recherche rapide)
        if (localDb) {
            // Index corrigé : inclut tous les champs utilisés dans le selector (type, title) et le champ de tri (likes).
            // L'ordre doit respecter les contraintes de Mango Indexing.
            await localDb.createIndex({
                index: { fields: ['type', 'title', 'likes'] }
            });
            console.log('✅ Index de recherche (Mango) créés avec succès pour le tri.');
        }

        // Démarrage de la réplication et chargement initial
        startReplication();
        await fetchDocuments(); 

    } catch (err) {
        console.error('❌ Erreur Init:', err);
        searchStatus.value = "Erreur de connexion à la base de données. Vérifiez CouchDB et l'URL.";
    }
};

// --- 2. Réplication Bidirectionnelle Live ---
const startReplication = () => {
    if (!localDb || !remoteDb) return;
    if (syncHandler) syncHandler.cancel(); // Annule la sync précédente si elle existe

    syncHandler = PouchDB.sync(localDb, remoteDb, { 
        live: true,    // Garde la réplication active
        retry: true    // Tente de se reconnecter en cas d'erreur ou de déconnexion
    })
    .on('change', (info) => {
        // Appelé lorsqu'une modification arrive du serveur (ou localement)
        console.log('🔄 Changement détecté et synchronisé.');
        // Si on n'est pas en train de chercher, on rafraîchit la liste pour montrer les changements
        if (searchQuery.value === '') {
            fetchDocuments(); 
        }
    })
    .on('error', (err) => {
        console.error('❌ Erreur Sync:', err);
    });
    
    console.log('✅ Synchronisation bidirectionnelle DÉMARRÉE.');
}

// --- 3. CRUD : Create Massif (Factory) ---
const generateMassData = async () => {
    if (!localDb) return;
    const posts: Post[] = [];
    // Génère 20 documents d'un coup pour tester la recherche et le tri
    for (let i = 1; i <= 20; i++) {
        posts.push(postFactory(i));
    }
    try {
        await localDb.bulkDocs(posts); // bulkDocs est la méthode la plus efficace
        console.log('✅ 20 documents générés et synchronisés !');
        if (searchQuery.value === '') fetchDocuments();
    } catch (err) {
        console.error('❌ Erreur mass factory:', err);
    }
};

// --- 4. CRUD : Create Unitaire ---
const createNewPost = async () => {
    if (!localDb) return;
    try {
        await localDb.put(postFactory(Math.floor(Math.random() * 1000) + 1));
        console.log('✅ Post créé.');
        if (searchQuery.value === '') fetchDocuments();
    } catch (err) { console.error(err); }
}

// --- 5. CRUD : Read (Lecture standard de tout) ---
const fetchDocuments = async () => {
    if (!localDb) return;
    try {
        // allDocs récupère tous les documents, triés par ID décroissant par défaut
        const result = await localDb.allDocs({ include_docs: true, descending: true });
        documents.value = result.rows
            .map(row => row.doc as Post)
            .filter(doc => doc.type === 'Post'); // Filtrage par type
        searchStatus.value = `${documents.value.length} documents affichés (Tout).`;
    } catch (err) { console.error(err); }
}

// --- 6. RECHERCHE AVANCÉE (Utilisation de PouchDB-Find) ---
const searchDocuments = async () => {
    if (!localDb) return;
    const query = searchQuery.value.trim();

    // Si le champ est vide, on revient à l'affichage de tous les documents
    if (query === '') {
        await fetchDocuments();
        return;
    }

    console.log(`🔎 Recherche indexée pour : "${query}"...`);
    try {
        // .find() utilise l'index Mango que nous avons créé au démarrage
        const result = await localDb.find({
            selector: {
                type: 'Post',
                // Utilisation d'une expression régulière pour une recherche "LIKE" insensible à la casse
                title: { $regex: RegExp(query, "i") } 
            },
            // Correction du tri : on trie par 'likes' et on doit inclure tous les champs indexés utilisés.
            sort: [
                { 'type': 'desc' }, // Doit correspondre à l'index
                { 'title': 'desc' }, // Doit correspondre à l'index
                { 'likes': 'desc' } // Doit correspondre à l'index
            ] 
        });

        // Mise à jour des documents affichés
        documents.value = result.docs as Post[];
        searchStatus.value = `🔍 ${documents.value.length} résultat(s) trouvé(s) pour "${query}", trié(s) par Likes.`;

    } catch (err) {
        // L'erreur de l'indexation sera maintenant journalisée, mais l'application devrait fonctionner.
        // Si l'erreur persiste, c'est que l'index n'est pas encore prêt, mais le retry de l'initDb devrait corriger.
        console.error('❌ Erreur de recherche (PouchDB-Find):', err);
        searchStatus.value = "Erreur de recherche. Assurez-vous que les index sont bien créés.";
    }
}

// --- 7. Update (Liker) ---
const likePost = async (post: Post) => {
    if (!localDb) return;
    try {
        // 1. Créer une nouvelle révision
        const updated = { ...post, likes: post.likes + 1 } as Post;
        // 2. Sauvegarder (Update)
        await localDb.put(updated);
        
        // 3. Mise à jour optimiste de l'interface pour fluidité (si le document est visible)
        const index = documents.value.findIndex(d => d._id === post._id);
        if (index !== -1) documents.value[index] = updated; 
        
        // Si une recherche/tri est en cours, il faut relancer la recherche pour mettre à jour l'ordre
        if (searchQuery.value !== '') {
            searchDocuments();
        }

    } catch (err) { console.error(err); }
}

// --- 8. Delete ---
const deletePost = async (post: Post) => {
    if (!localDb) return;
    try {
        // Suppression nécessite l'ID et le dernier _rev
        await localDb.remove(post._id, post._rev!);
        // Retirer de la liste locale immédiatement
        documents.value = documents.value.filter(d => d._id !== post._id);
    } catch (err) { console.error(err); }
}

// --- 9. Toggle Offline Mode ---
const toggleOffline = () => {
    isOnline.value = !isOnline.value;
    if (isOnline.value) {
        // Repasse en mode ONLINE
        startReplication();
        console.log('🌐 Passage en mode ONLINE. Synchronisation reprise.');
    } else {
        // Passe en mode OFFLINE
        if (syncHandler) syncHandler.cancel();
        console.log('📴 Passage en mode OFFLINE. Synchronisation annulée.');
    }
}

// Surveille la saisie de l'utilisateur pour lancer la recherche
watch(searchQuery, () => {
    searchDocuments();
});

// Appelé au chargement du composant
onMounted(() => {
    initDatabase();
});
</script>

<template>
  <div class="container">
    <header>
        <h1><i class="icon-database"></i> InfraDon 2 - PouchDB/CouchDB</h1>
        <div class="status-bar">
            <!-- Affichage du statut de la connexion / sync -->
            <span class="status" :class="{ online: isOnline, offline: !isOnline }">
                {{ isOnline ? '🟢 ONLINE (Sync Live)' : '🔴 OFFLINE (Local Only)' }}
            </span>
            <button @click="toggleOffline" class="btn-small">Basculer Mode (Simuler)</button>
        </div>
    </header>

    <section class="controls">
        <div class="card control-box create-box">
            <h3>1. Création de Documents</h3>
            <div class="buttons">
                <button @click="createNewPost" class="btn-primary">➕ Créer 1 Post</button>
                <button @click="generateMassData" class="btn-secondary">🏭 Factory (20 Posts)</button>
            </div>
        </div>

        <div class="card control-box search-box">
            <h3>2. Recherche Indexée & Tri</h3>
            <!-- Champ de recherche qui déclenche la méthode searchDocuments via le watch() -->
            <input 
                v-model="searchQuery" 
                type="text" 
                placeholder="🔍 Rechercher un titre..."
                class="search-input"
            />
            <!-- Message de statut : nombre de résultats et méthode de tri -->
            <p class="search-info">{{ searchStatus }}</p>
        </div>
    </section>

    <section class="results">
        <h2>Liste des Posts</h2>
        <!-- État vide si aucun document n'est affiché -->
        <div v-if="documents.length === 0" class="empty-state">
            <i class="icon-empty"></i>
            Aucun document trouvé. Créez des posts pour démarrer ou vérifiez votre recherche.
        </div>
        <!-- Grille des documents -->
        <div class="grid">
            <div v-for="doc in documents" :key="doc._id" class="post-card">
                <div class="post-header">
                    <span class="post-title">{{ doc.title }}</span>
                    <!-- Affichage de l'attribut 'likes' -->
                    <span class="likes"><i class="icon-like"></i> {{ doc.likes }}</span>
                </div>
                <p class="post-content">{{ doc.content }}</p>
                <div class="actions">
                    <!-- Bouton pour liker (CRUD - Update) -->
                    <button @click="likePost(doc)" class="btn-like">👍 J'aime</button>
                    <!-- Bouton pour supprimer (CRUD - Delete) -->
                    <button @click="deletePost(doc)" class="btn-danger">🗑️ Supprimer</button>
                </div>
                <small class="id-text">ID: {{ doc._id.substring(0, 15) }}...</small>
            </div>
        </div>
    </section>
  </div>
</template>

<style scoped>
/*
 * Styles Améliorés
 * Utilise une palette plus douce et des ombres subtiles pour un look moderne.
 */

:root {
    --color-primary: #1e90ff; /* Bleu éclatant */
    --color-secondary: #ffac33; /* Orange doux pour l'usine */
    --color-success: #28a745;
    --color-danger: #dc3545;
    --color-background: #111c24ff;
    --color-card-bg: #ffffff;
    --shadow-light: 0 4px 10px rgba(0, 0, 0, 0.05);
    --shadow-hover: 0 8px 15px rgba(0, 0, 0, 0.1);
    --border-radius: 12px;
}

body {
    background-color: var(--color-background);
}

.container { 
    font-family: 'Inter', 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; 
    max-width: 960px; 
    margin: 0 auto; 
    padding: 30px 20px; 
    color: #2c3e50; 
}

/* --- HEADER / TITRE --- */
header { 
    display: flex; 
    flex-direction: column; 
    align-items: center; 
    text-align: center;
    border-bottom: 2px solid #e0e0e0; 
    margin-bottom: 30px; 
    padding-bottom: 15px; 
}
h1 { 
    font-size: 2em; 
    color: var(--color-primary); 
    margin-bottom: 15px; 
    font-weight: 700;
}
.icon-database { 
    /* Utilisation d'un caractère Unicode simple pour l'icône, ou SVG si disponible */
    content: '💾'; 
    margin-right: 8px; 
    font-size: 1.1em;
}

/* --- STATUS BAR --- */
.status-bar { 
    display: flex; 
    gap: 15px; 
    align-items: center; 
    padding: 8px 15px;
    border-radius: var(--border-radius);
    background: var(--color-card-bg);
    box-shadow: var(--shadow-light);
}
.status { 
    padding: 5px 12px; 
    border-radius: 20px; 
    font-weight: bold; 
    font-size: 0.85em; 
    transition: all 0.3s; 
    text-transform: uppercase;
}
.online { 
    background: #d4edda; 
    color: var(--color-success); 
}
.offline { 
    background: #f8d7da; 
    color: var(--color-danger); 
}

/* --- CONTROLS & CARDS --- */
.controls { 
    display: grid; 
    grid-template-columns: 1fr 2fr; 
    gap: 20px; 
    margin-bottom: 30px; 
}
@media (max-width: 768px) {
    .controls { grid-template-columns: 1fr; }
    .create-box { order: 2; }
    .search-box { order: 1; }
}

.card { 
    background: var(--color-card-bg); 
    padding: 20px; 
    border-radius: var(--border-radius); 
    border: 1px solid #e9ecef; 
    box-shadow: var(--shadow-light); 
    transition: box-shadow 0.3s;
}
.card h3 {
    margin-top: 0;
    color: var(--color-primary);
    font-size: 1.2em;
    border-bottom: 1px dashed #e9ecef;
    padding-bottom: 10px;
}

.buttons { display: flex; gap: 10px; margin-top: 15px; }

.search-input { 
    width: 100%; 
    padding: 12px; 
    border: 2px solid #ccc; 
    border-radius: 8px; 
    margin-top: 10px; 
    font-size: 1em; 
    transition: border-color 0.3s, box-shadow 0.3s;
}
.search-input:focus {
    border-color: var(--color-primary);
    box-shadow: 0 0 0 3px rgba(30, 144, 255, 0.2);
    outline: none;
}
.search-info { 
    font-size: 0.8em; 
    color: #6c757d; 
    margin-top: 8px; 
    font-style: italic; 
}

/* --- GRID & POST CARDS --- */
h2 {
    color: #495057;
    border-bottom: 1px solid #e0e0e0;
    padding-bottom: 10px;
    margin-bottom: 20px;
}
.grid { 
    display: grid; 
    grid-template-columns: repeat(auto-fill, minmax(300px, 1fr)); 
    gap: 20px; 
}
.post-card { 
    padding: 20px; 
    border-radius: var(--border-radius); 
    background: var(--color-card-bg); 
    box-shadow: var(--shadow-light); 
    transition: transform 0.2s, box-shadow 0.2s; 
}
.post-card:hover { 
    transform: translateY(-5px); 
    box-shadow: var(--shadow-hover); 
}

.post-header { 
    display: flex; 
    justify-content: space-between; 
    align-items: center; 
    margin-bottom: 15px; 
    font-size: 1.2em; 
    border-bottom: 1px solid #f8f9fa; 
    padding-bottom: 5px;
}
.post-title {
    font-weight: 600;
    color: #333;
}
.likes { 
    color: #e91e63; /* Rose pour les likes */
    font-weight: 700; 
    font-size: 1.1em; 
    display: flex;
    align-items: center;
    gap: 4px;
}
.icon-like {
    content: '❤️';
    font-size: 1.1em;
}

.post-content {
    color: #555;
    line-height: 1.4;
    margin-bottom: 15px;
}
.actions { 
    margin-top: 15px; 
    display: flex; 
    gap: 10px; 
    justify-content: flex-end; 
    border-top: 1px solid #f8f9fa;
    padding-top: 10px;
}
.id-text { 
    display: block; 
    margin-top: 10px; 
    font-size: 0.75em; 
    color: #aaa; 
    text-align: right; 
}
.empty-state { 
    text-align: center; 
    padding: 50px; 
    color: #888; 
    background: #e9ecef; 
    border-radius: var(--border-radius); 
    grid-column: 1 / -1; 
    border: 2px dashed #ced4da; 
    font-size: 1.1em;
}
.icon-empty {
    content: '📄';
    display: block;
    font-size: 2em;
    margin-bottom: 10px;
}


/* --- BOUTONS UNIVERSELS --- */
button { 
    cursor: pointer; 
    padding: 10px 18px; 
    border: none; 
    border-radius: 8px; 
    font-weight: 600; 
    transition: background 0.2s, transform 0.1s, box-shadow 0.2s; 
    box-shadow: 0 2px 5px rgba(0,0,0,0.1);
}
button:active { 
    transform: scale(0.98); 
    box-shadow: none;
}

.btn-primary { 
    background: var(--color-primary); 
    color: white; 
}
.btn-primary:hover { 
    background: #106fcc; 
}

.btn-secondary { 
    background: var(--color-secondary); 
    color: #333; 
}
.btn-secondary:hover { 
    background: #e69500; 
}

.btn-like { 
    background: #ffe3e9; 
    color: #e91e63; 
}
.btn-like:hover { 
    background: #ffcdd2; 
}

.btn-danger { 
    background: var(--color-danger); 
    color: white; 
}
.btn-danger:hover { 
    background: #c82333; 
}

.btn-small { 
    font-size: 0.85em; 
    padding: 6px 12px; 
    background: #6c757d; 
    color: white; 
    box-shadow: none;
}
.btn-small:hover { 
    background: #5a6268; 
}
</style>