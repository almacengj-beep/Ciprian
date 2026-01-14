# Ciprian
Inspección agencias nuevas
import React, { useState, useEffect } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from 'firebase/auth';
import { getFirestore, collection, addDoc, serverTimestamp } from 'firebase/firestore';
import { 
  ClipboardCheck, 
  Zap, 
  Building, 
  Monitor, 
  Network, 
  Camera, 
  Save, 
  Printer, 
  FileText,
  Sparkles,
  Loader2,
  MapPin,
  CheckCircle2
} from 'lucide-react';

// === CONFIGURACIÓN DINÁMICA ===
// Detectamos si estamos en el entorno de Canvas o en producción propia
const getFirebaseConfig = () => {
  try {
    if (typeof __firebase_config !== 'undefined') {
      return JSON.parse(__firebase_config);
    }
  } catch (e) {
    console.error("Error cargando configuración automática:", e);
  }
  
  // Valores por defecto para cuando lo lleves a tu propio hosting
  return {
    apiKey: "TU_API_KEY_REAL",
    authDomain: "TU_PROYECTO.firebaseapp.com",
    projectId: "TU_PROYECTO_ID",
    storageBucket: "TU_PROYECTO.appspot.com",
    messagingSenderId: "TU_SENDER_ID",
    appId: "TU_APP_ID"
  };
};

const firebaseConfig = getFirebaseConfig();
const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);
const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';

const App = () => {
  const [user, setUser] = useState(null);
  const [isSubmitting, setIsSubmitting] = useState(false);
  const [saveSuccess, setSaveSuccess] = useState(false);
  
  const [generalData, setGeneralData] = useState({
    sucursal: '',
    codigoSucursal: '',
    fecha: new Date().toISOString().split('T')[0],
    inspector: '',
  });
  
  const [location, setLocation] = useState(null);
  const [isLocating, setIsLocating] = useState(false);
  const [locationError, setLocationError] = useState(null);

  const [sections, setSections] = useState({
    electrica: { rating: '', details: '' },
    infraestructura: { rating: '', details: '' },
    informatica: { rating: '', details: '' },
    red: { rating: '', details: '' }
  });

  const [summary, setSummary] = useState('');

  // Autenticación siguiendo la Regla 3 de Firestore
  useEffect(() => {
    const initAuth = async () => {
      try {
        if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
          await signInWithCustomToken(auth, __initial_auth_token);
        } else {
          await signInAnonymously(auth);
        }
      } catch (err) {
        console.error("Error Auth:", err);
      }
    };

    initAuth();
    const unsubscribe = onAuthStateChanged(auth, (u) => setUser(u));
    return () => unsubscribe();
  }, []);

  const handleGetLocation = () => {
    if (!navigator.geolocation) return setLocationError("GPS no disponible");
    setIsLocating(true);
    setLocationError(null);
    
    navigator.geolocation.getCurrentPosition(
      (pos) => {
        setLocation({ lat: pos.coords.latitude.toFixed(6), lng: pos.coords.longitude.toFixed(6) });
        setIsLocating(false);
      },
      (err) => {
        setLocationError("Error GPS: Asegúrate de dar permisos en el navegador.");
        setIsLocating(false);
      },
      { enableHighAccuracy: true, timeout: 15000 }
    );
  };

  const handleSaveData = async () => {
    if (!user) return alert("Esperando conexión con el servidor...");
    setIsSubmitting(true);
    
    try {
      // Usamos la ruta estricta requerida por el entorno (Regla 1)
      const targetPath = collection(db, 'artifacts', appId, 'public', 'data', 'inspecciones');
      
      await addDoc(targetPath, {
        ...generalData,
        secciones: sections,
        resumenIA: summary,
        ubicacion: location,
        userId: user.uid,
        timestamp: serverTimestamp()
      });
      
      setSaveSuccess(true);
      setTimeout(() => setSaveSuccess(false), 3000);
      alert("¡Inspección guardada exitosamente en la nube!");
    } catch (error) {
      console.error("Error guardando:", error);
      alert("Error al guardar: " + error.message);
    } finally {
      setIsSubmitting(false);
    }
  };

  return (
    <div className="min-h-screen bg-slate-50 p-2 md:p-8 font-sans">
      <div className="max-w-xl mx-auto bg-white rounded-2xl shadow-lg overflow-hidden pb-24 md:pb-8">
        <header className="bg-slate-900 text-white p-5">
          <h1 className="text-xl font-bold flex items-center gap-2">
            <ClipboardCheck className="text-blue-400" /> Inspección de Agencia
          </h1>
          <div className="flex justify-between items-center mt-1">
            <p className="text-[10px] text-slate-400">ID Usuario: {user?.uid || 'Conectando...'}</p>
            {saveSuccess && <span className="text-[10px] bg-green-500 text-white px-2 py-0.5 rounded-full animate-pulse">Sincronizado</span>}
          </div>
        </header>

        <main className="p-4 space-y-6">
          {/* Datos de Identificación */}
          <section className="space-y-3 bg-white p-2 rounded-xl border border-slate-100 shadow-sm">
            <label className="text-[10px] font-bold text-slate-400 uppercase ml-1">Información General</label>
            <input 
              type="text" 
              placeholder="Nombre de la Agencia" 
              className="w-full p-3 border rounded-xl bg-slate-50 focus:bg-white outline-none transition-all" 
              onChange={e => setGeneralData({...generalData, sucursal: e.target.value})} 
            />
            <input 
              type="text" 
              placeholder="Código de Sucursal" 
              className="w-full p-3 border rounded-xl bg-slate-50 focus:bg-white outline-none transition-all" 
              onChange={e => setGeneralData({...generalData, codigoSucursal: e.target.value})} 
            />
            
            <button 
              type="button"
              onClick={handleGetLocation}
              disabled={isLocating}
              className={`w-full p-3 rounded-xl flex items-center justify-center gap-2 font-bold transition-all ${location ? 'bg-green-50 text-green-700 border border-green-200' : 'bg-slate-800 text-white'}`}
            >
              {isLocating ? <Loader2 className="animate-spin w-4 h-4" /> : <MapPin size={18} />}
              {location ? `GPS: ${location.lat}, ${location.lng}` : "Capturar Ubicación GPS"}
            </button>
            {locationError && <p className="text-[10px] text-red-500 text-center font-bold uppercase">{locationError}</p>}
          </section>

          {/* Categorías Técnicas */}
          {['electrica', 'infraestructura', 'informatica', 'red'].map((key) => (
            <div key={key} className="border border-slate-200 rounded-2xl p-4 bg-white shadow-sm">
              <h3 className="text-xs font-bold uppercase mb-3 flex items-center gap-2 text-slate-700">
                <div className="w-1.5 h-1.5 rounded-full bg-blue-500"></div> 
                {key === 'electrica' ? 'Sistema Eléctrico' : 
                 key === 'infraestructura' ? 'Infraestructura' : 
                 key === 'informatica' ? 'Equipos IT' : 'Red y Datos'}
              </h3>
              <div className="flex gap-2 mb-3">
                {['Bueno', 'Malo'].map(r => (
                  <button 
                    key={r}
                    type="button"
                    onClick={() => setSections({...sections, [key]: {...sections[key], rating: r}})}
                    className={`flex-1 p-3 rounded-xl text-xs font-bold border transition-all ${sections[key].rating === r ? 'bg-blue-600 text-white border-blue-600 shadow-md scale-105' : 'bg-slate-50 text-slate-400 border-slate-100'}`}
                  >
                    {r}
                  </button>
                ))}
              </div>
              <textarea 
                placeholder="Añadir observaciones específicas..." 
                className="w-full p-3 text-sm border border-slate-100 rounded-xl bg-slate-50 focus:bg-white outline-none transition-all"
                rows="2"
                onChange={e => setSections({...sections, [key]: {...sections[key], details: e.target.value}})}
              ></textarea>
            </div>
          ))}
        </main>

        {/* Botón Flotante para Móvil */}
        <div className="fixed bottom-0 left-0 right-0 p-4 bg-white/80 backdrop-blur-md border-t border-slate-100 md:relative md:border-none md:bg-transparent">
          <button 
            type="button"
            onClick={handleSaveData}
            disabled={isSubmitting}
            className="w-full bg-blue-700 hover:bg-blue-800 text-white font-bold py-4 rounded-2xl shadow-xl flex items-center justify-center gap-3 active:scale-95 transition-all disabled:bg-slate-400"
          >
            {isSubmitting ? <Loader2 className="animate-spin" /> : <Save />}
            FINALIZAR Y GUARDAR
          </button>
        </div>
      </div>
    </div>
  );
};

export default App;
