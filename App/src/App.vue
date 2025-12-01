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
// Remplacez 'infra_53_0850' par le nom de votre base de données CouchDB
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
        // On indexe sur 'type' pour le filtrage et 'title' et 'likes' pour la recherche/tri.
        if (localDb) {
            await localDb.createIndex({
                index: { fields: ['type', 'title', 'likes'] }
            });
            console.log('✅ Index de recherche (Mango) créés avec succès.');
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
            // On trie les résultats par likes (du plus liké au moins liké)
            sort: [
                { 'type': 'desc' }, 
                { 'likes': 'desc' } // Tri par likes (du plus grand au plus petit)
            ] 
        });

        // Mise à jour des documents affichés
        documents.value = result.docs as Post[];
        searchStatus.value = `🔍 ${documents.value.length} résultat(s) trouvé(s) pour "${query}", trié(s) par Likes.`;

    } catch (err) {
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
        <h1>InfraDon 2 - Rendu Intermédiaire PouchDB/CouchDB</h1>
        <div class="status-bar">
            <!-- Affichage du statut de la connexion / sync -->
            <span class="status" :class="{ online: isOnline, offline: !isOnline }">
                {{ isOnline ? '🟢 ONLINE (Sync Active)' : '🔴 OFFLINE (Local DB)' }}
            </span>
            <button @click="toggleOffline" class="btn-small">Basculer Mode</button>
        </div>
    </header>

    <section class="controls">
        <div class="card">
            <h3>1. Création de Données (CRUD)</h3>
            <div class="buttons">
                <button @click="createNewPost" class="btn-primary">➕ Ajouter 1 Post</button>
                <button @click="generateMassData" class="btn-warning">🏭 Factory (20 Posts)</button>
            </div>
        </div>

        <div class="card">
            <h3>2. Recherche & Tri (Indexé)</h3>
            <!-- Champ de recherche qui déclenche la méthode searchDocuments via le watch() -->
            <input 
                v-model="searchQuery" 
                type="text" 
                placeholder="🔍 Rechercher un titre (ex: Message #5)..."
                class="search-input"
            />
            <!-- Message de statut : nombre de résultats et méthode de tri -->
            <p class="search-info">{{ searchStatus }}</p>
        </div>
    </section>

    <section class="results">
        <h2>Liste des Documents</h2>
        <!-- État vide si aucun document n'est affiché -->
        <div v-if="documents.length === 0" class="empty-state">
            Aucun document trouvé. Créez des posts ou modifiez votre recherche.
        </div>
        <!-- Grille des documents -->
        <div class="grid">
            <div v-for="doc in documents" :key="doc._id" class="post-card">
                <div class="post-header">
                    <strong>{{ doc.title }}</strong>
                    <!-- Affichage de l'attribut 'likes' -->
                    <span class="likes">❤️ {{ doc.likes }}</span>
                </div>
                <p>{{ doc.content }}</p>
                <div class="actions">
                    <!-- Bouton pour liker (CRUD - Update) -->
                    <button @click="likePost(doc)">👍 Like</button>
                    <!-- Bouton pour supprimer (CRUD - Delete) -->
                    <button @click="deletePost(doc)" class="btn-danger">🗑️ Suppr</button>
                </div>
                <small class="id-text">ID Local: {{ doc._id.substring(0, 10) }}...</small>
            </div>
        </div>
    </section>
  </div>
</template>

<style scoped>
/* Styles de base pour l'esthétique et la réactivité */
.container { font-family: 'Segoe UI', sans-serif; max-width: 900px; margin: 0 auto; padding: 20px; color: #333; }
header { display: flex; flex-direction: column; align-items: center; border-bottom: 2px solid #eee; margin-bottom: 20px; padding-bottom: 10px; }
h1 { font-size: 1.8em; color: #007bff; margin-bottom: 10px; }

.status-bar { display: flex; gap: 10px; align-items: center; }
.status { padding: 5px 10px; border-radius: 15px; font-weight: bold; font-size: 0.9em; transition: all 0.3s; }
.online { background: #d4edda; color: #155724; box-shadow: 0 0 5px rgba(21, 87, 36, 0.5); }
.offline { background: #f8d7da; color: #721c24; box-shadow: 0 0 5px rgba(114, 28, 36, 0.5); }

.controls { display: grid; grid-template-columns: 1fr 1fr; gap: 20px; margin-bottom: 20px; }
@media (max-width: 600px) {
    .controls { grid-template-columns: 1fr; }
}

.card { background: #f9f9f9; padding: 15px; border-radius: 8px; border: 1px solid #ddd; box-shadow: 0 1px 3px rgba(0,0,0,0.08); }
.buttons { display: flex; gap: 10px; margin-top: 10px; }
.search-input { width: 100%; padding: 10px; border: 2px solid #ccc; border-radius: 6px; margin-top: 10px; font-size: 1em; }
.search-info { font-size: 0.85em; color: #666; margin-top: 5px; font-style: italic; }

.grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(280px, 1fr)); gap: 15px; }
.post-card { border: 1px solid #e0e0e0; padding: 15px; border-radius: 8px; background: white; box-shadow: 0 2px 5px rgba(0,0,0,0.05); transition: transform 0.2s, box-shadow 0.2s; }
.post-card:hover { transform: translateY(-3px); box-shadow: 0 4px 10px rgba(0,0,0,0.1); }

.post-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 10px; font-size: 1.1em; border-bottom: 1px solid #eee; padding-bottom: 5px;}
.likes { color: #e91e63; font-weight: bold; font-size: 1.2em; }
.actions { margin-top: 15px; display: flex; gap: 10px; justify-content: flex-end; }
.id-text { display: block; margin-top: 10px; font-size: 0.7em; color: #aaa; text-align: left; }
.empty-state { text-align: center; padding: 40px; color: #888; background: #f9f9f9; border-radius: 8px; grid-column: 1 / -1; border: 2px dashed #ddd; }

button { cursor: pointer; padding: 8px 15px; border: none; border-radius: 6px; font-weight: 500; transition: background 0.2s, transform 0.1s; }
button:active { transform: scale(0.98); }

.btn-primary { background: #007bff; color: white; box-shadow: 0 2px 4px rgba(0, 123, 255, 0.4); }
.btn-primary:hover { background: #0056b3; }
.btn-warning { background: #ffc107; color: #333; box-shadow: 0 2px 4px rgba(255, 193, 7, 0.4); }
.btn-warning:hover { background: #e0a800; }
.btn-danger { background: #dc3545; color: white; box-shadow: 0 2px 4px rgba(220, 53, 69, 0.4); }
.btn-danger:hover { background: #c82333; }
.btn-small { font-size: 0.8em; padding: 5px 10px; background: #6c757d; color: white; }
.btn-small:hover { background: #5a6268; }
</style>