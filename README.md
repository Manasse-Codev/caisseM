# caisseM
Je vais te détailler **chaque couche technique** de l'architecture, avec du **code concret** et des **exemples fonctionnels** pour une application de gestion de trésorerie/caisse.

## 📁 Structure complète du projet

```
gestion_caisse/
│
├── backend/                      # CŒUR MÉTIER (partagé)
│   ├── models/                   # SQLAlchemy
│   │   ├── __init__.py
│   │   ├── database.py          # Connexion DB
│   │   ├── mouvement.py         # Mouvement de caisse
│   │   ├── caisse.py            # Comptes/caisses
│   │   └── utilisateur.py       # Utilisateurs
│   │
│   ├── services/                 # LOGIQUE MÉTIER
│   │   ├── __init__.py
│   │   ├── caisse_service.py    # Encaissement/décaissement
│   │   ├── tresorerie_service.py # Prévisionnel, rapports
│   │   └── export_service.py    # PDF, Excel
│   │
│   ├── api/                      # API REST (pour version web)
│   │   ├── __init__.py
│   │   ├── main.py              # FastAPI app
│   │   ├── routes/
│   │   │   ├── mouvements.py
│   │   │   ├── caisses.py
│   │   │   └── auth.py
│   │   └── schemas.py           # Pydantic models
│   │
│   └── utils/                    # UTILITAIRES
│       ├── security.py          # Hash, JWT
│       └── validators.py        # Validation entrées
│
├── desktop/                      # VERSION BUREAU
│   ├── main.py                  # Entry point
│   ├── windows/
│   │   ├── main_window.py       # Fenêtre principale Tkinter
│   │   ├── encaissement.py
│   │   └── rapports.py
│   └── assets/                  # Icônes, styles
│
├── web/                          # VERSION WEB
│   ├── frontend/                # React/Vue (à part)
│   └── docker/                  # Dockerfile, nginx
│
├── requirements.txt
└── README.md
```

---

## 1️⃣ COUCHE MODÈLES (Base de données)

### `backend/models/database.py`
```python
from sqlalchemy import create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker
import os

# Choix DB selon environnement
DB_TYPE = os.getenv("DB_TYPE", "sqlite")  # sqlite ou postgresql

if DB_TYPE == "sqlite":
    DATABASE_URL = "sqlite:///./caisse.db"
    engine = create_engine(DATABASE_URL, connect_args={"check_same_thread": False})
else:
    DATABASE_URL = os.getenv("DATABASE_URL", "postgresql://user:pass@localhost/caisse")
    engine = create_engine(DATABASE_URL)

SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
Base = declarative_base()

# Dépendance pour obtenir session DB
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

### `backend/models/mouvement.py`
```python
from sqlalchemy import Column, Integer, String, Float, DateTime, ForeignKey, Enum
from sqlalchemy.orm import relationship
from datetime import datetime
import enum
from .database import Base

class TypeMouvement(str, enum.Enum):
    ENCAISSEMENT = "encaissement"
    DECAISSEMENT = "decaissement"

class ModePaiement(str, enum.Enum):
    ESPECES = "espèces"
    CARTE = "carte bancaire"
    CHEQUE = "chèque"
    VIREMENT = "virement"

class Mouvement(Base):
    __tablename__ = "mouvements"
    
    id = Column(Integer, primary_key=True, index=True)
    date = Column(DateTime, default=datetime.now)
    type_mouvement = Column(Enum(TypeMouvement), nullable=False)
    montant = Column(Float, nullable=False)
    mode_paiement = Column(Enum(ModePaiement), nullable=False)
    description = Column(String(255))
    reference = Column(String(100), unique=True)  # N° facture, chèque, etc.
    
    # Clé étrangère vers la caisse
    caisse_id = Column(Integer, ForeignKey("caisses.id"))
    caisse = relationship("Caisse", back_populates="mouvements")
    
    # Utilisateur qui a saisi
    utilisateur_id = Column(Integer, ForeignKey("utilisateurs.id"))
    utilisateur = relationship("Utilisateur")
```

### `backend/models/caisse.py`
```python
from sqlalchemy import Column, Integer, String, Float
from sqlalchemy.orm import relationship
from .database import Base

class Caisse(Base):
    __tablename__ = "caisses"
    
    id = Column(Integer, primary_key=True, index=True)
    nom = Column(String(50), unique=True, nullable=False)  # "Caisse principale", "Caisse 2"
    solde_initial = Column(Float, default=0)
    
    # Relations
    mouvements = relationship("Mouvement", back_populates="caisse")
    
    @property
    def solde_actuel(self):
        """Calcul dynamique du solde"""
        total_encaissements = sum(m.montant for m in self.mouvements 
                                 if m.type_mouvement == TypeMouvement.ENCAISSEMENT)
        total_decaissements = sum(m.montant for m in self.mouvements 
                                 if m.type_mouvement == TypeMouvement.DECAISSEMENT)
        return self.solde_initial + total_encaissements - total_decaissements
```

---

## 2️⃣ COUCHE SERVICES (Logique métier)

### `backend/services/caisse_service.py`
```python
from sqlalchemy.orm import Session
from datetime import datetime, date
from typing import List, Dict
from ..models.mouvement import Mouvement, TypeMouvement, ModePaiement
from ..models.caisse import Caisse

class CaisseService:
    
    def __init__(self, db: Session):
        self.db = db
    
    def encaisser(self, montant: float, mode: ModePaiement, description: str, 
                  caisse_id: int, utilisateur_id: int, reference: str = None) -> Mouvement:
        """Enregistrer un encaissement"""
        if montant <= 0:
            raise ValueError("Le montant doit être positif")
        
        mouvement = Mouvement(
            type_mouvement=TypeMouvement.ENCAISSEMENT,
            montant=montant,
            mode_paiement=mode,
            description=description,
            reference=reference,
            caisse_id=caisse_id,
            utilisateur_id=utilisateur_id
        )
        
        self.db.add(mouvement)
        self.db.commit()
        self.db.refresh(mouvement)
        return mouvement
    
    def decaisser(self, montant: float, mode: ModePaiement, description: str,
                  caisse_id: int, utilisateur_id: int, reference: str = None) -> Mouvement:
        """Enregistrer un décaissement avec vérification de solde"""
        if montant <= 0:
            raise ValueError("Le montant doit être positif")
        
        # Vérifier solde suffisant
        caisse = self.db.query(Caisse).filter(Caisse.id == caisse_id).first()
        if caisse.solde_actuel < montant:
            raise ValueError(f"Solde insuffisant. Solde actuel: {caisse.solde_actuel}€")
        
        mouvement = Mouvement(
            type_mouvement=TypeMouvement.DECAISSEMENT,
            montant=montant,
            mode_paiement=mode,
            description=description,
            reference=reference,
            caisse_id=caisse_id,
            utilisateur_id=utilisateur_id
        )
        
        self.db.add(mouvement)
        self.db.commit()
        self.db.refresh(mouvement)
        return mouvement
    
    def get_solde_journalier(self, caisse_id: int, date_jour: date = None) -> Dict:
        """Obtenir solde du jour + récap"""
        if date_jour is None:
            date_jour = date.today()
        
        mouvements = self.db.query(Mouvement).filter(
            Mouvement.caisse_id == caisse_id,
            Mouvement.date >= date_jour,
            Mouvement.date < date_jour.replace(day=date_jour.day+1)
        ).all()
        
        total_encaissements = sum(m.montant for m in mouvements 
                                 if m.type_mouvement == TypeMouvement.ENCAISSEMENT)
        total_decaissements = sum(m.montant for m in mouvements 
                                 if m.type_mouvement == TypeMouvement.DECAISSEMENT)
        
        # Récupérer solde de début de journée
        mouvements_anterieurs = self.db.query(Mouvement).filter(
            Mouvement.caisse_id == caisse_id,
            Mouvement.date < date_jour
        ).all()
        
        caisse = self.db.query(Caisse).filter(Caisse.id == caisse_id).first()
        solde_initial = caisse.solde_initial
        solde_fin_journee = solde_initial + total_encaissements - total_decaissements
        
        return {
            "date": date_jour,
            "solde_debut": solde_initial,
            "total_encaissements": total_encaissements,
            "total_decaissements": total_decaissements,
            "solde_fin": solde_fin_journee,
            "mouvements": mouvements
        }
    
    def get_rapport_mensuel(self, mois: int, annee: int, caisse_id: int = None) -> List[Dict]:
        """Rapport de trésorerie mensuel"""
        from sqlalchemy import extract
        
        query = self.db.query(Mouvement).filter(
            extract('month', Mouvement.date) == mois,
            extract('year', Mouvement.date) == annee
        )
        
        if caisse_id:
            query = query.filter(Mouvement.caisse_id == caisse_id)
        
        mouvements = query.all()
        
        # Regroupement par mode de paiement
        recap = {}
        for mode in ModePaiement:
            encaissements = sum(m.montant for m in mouvements 
                              if m.type_mouvement == TypeMouvement.ENCAISSEMENT 
                              and m.mode_paiement == mode)
            decaissements = sum(m.montant for m in mouvements 
                              if m.type_mouvement == TypeMouvement.DECAISSEMENT 
                              and m.mode_paiement == mode)
            recap[mode.value] = {"encaissements": encaissements, "decaissements": decaissements}
        
        return {
            "mois": mois,
            "annee": annee,
            "total_entrees": sum(m.montant for m in mouvements if m.type_mouvement == TypeMouvement.ENCAISSEMENT),
            "total_sorties": sum(m.montant for m in mouvements if m.type_mouvement == TypeMouvement.DECAISSEMENT),
            "solde_final": sum(m.montant for m in mouvements if m.type_mouvement == TypeMouvement.ENCAISSEMENT) -
                          sum(m.montant for m in mouvements if m.type_mouvement == TypeMouvement.DECAISSEMENT),
            "par_mode": recap
        }
```

### `backend/services/export_service.py`
```python
import pandas as pd
from reportlab.lib import colors
from reportlab.lib.pagesizes import A4
from reportlab.platypus import SimpleDocTemplate, Table, TableStyle, Paragraph
from reportlab.lib.styles import getSampleStyleSheet
from datetime import datetime
import io

class ExportService:
    
    @staticmethod
    def export_to_excel(mouvements: list, filename: str = None) -> bytes:
        """Exporter les mouvements vers Excel"""
        data = []
        for m in mouvements:
            data.append({
                "Date": m.date.strftime("%d/%m/%Y %H:%M"),
                "Type": m.type_mouvement.value,
                "Montant": m.montant,
                "Mode": m.mode_paiement.value,
                "Description": m.description,
                "Référence": m.reference or ""
            })
        
        df = pd.DataFrame(data)
        
        output = io.BytesIO()
        with pd.ExcelWriter(output, engine='openpyxl') as writer:
            df.to_excel(writer, sheet_name="Mouvements", index=False)
            
            # Formater les colonnes
            worksheet = writer.sheets["Mouvements"]
            for column in worksheet.columns:
                max_length = 0
                column_letter = column[0].column_letter
                for cell in column:
                    try:
                        if len(str(cell.value)) > max_length:
                            max_length = len(str(cell.value))
                    except:
                        pass
                adjusted_width = min(max_length + 2, 50)
                worksheet.column_dimensions[column_letter].width = adjusted_width
        
        return output.getvalue()
    
    @staticmethod
    def export_to_pdf(rapport_data: dict) -> bytes:
        """Exporter rapport vers PDF"""
        buffer = io.BytesIO()
        doc = SimpleDocTemplate(buffer, pagesize=A4)
        styles = getSampleStyleSheet()
        story = []
        
        # Titre
        title = Paragraph(f"Rapport de trésorerie - {rapport_data['mois']}/{rapport_data['annee']}", 
                         styles['Title'])
        story.append(title)
        
        # Tableau récapitulatif
        data_table = [["Indicateur", "Montant"]]
        data_table.append(["Total entrées", f"{rapport_data['total_entrees']:,.2f} €"])
        data_table.append(["Total sorties", f"{rapport_data['total_sorties']:,.2f} €"])
        data_table.append(["Solde final", f"{rapport_data['solde_final']:,.2f} €"])
        
        table = Table(data_table)
        table.setStyle(TableStyle([
            ('BACKGROUND', (0, 0), (-1, 0), colors.grey),
            ('TEXTCOLOR', (0, 0), (-1, 0), colors.whitesmoke),
            ('ALIGN', (0, 0), (-1, -1), 'CENTER'),
            ('FONTNAME', (0, 0), (-1, 0), 'Helvetica-Bold'),
            ('FONTSIZE', (0, 0), (-1, 0), 12),
            ('BOTTOMPADDING', (0, 0), (-1, 0), 12),
            ('GRID', (0, 0), (-1, -1), 1, colors.black)
        ]))
        story.append(table)
        
        doc.build(story)
        buffer.seek(0)
        return buffer.getvalue()
```

---

## 3️⃣ COUCHE API REST (Pour version web)

### `backend/api/schemas.py`
```python
from pydantic import BaseModel
from datetime import datetime
from enum import Enum

class TypeMouvementEnum(str, Enum):
    ENCAISSEMENT = "encaissement"
    DECAISSEMENT = "decaissement"

class ModePaiementEnum(str, Enum):
    ESPECES = "espèces"
    CARTE = "carte bancaire"
    CHEQUE = "chèque"
    VIREMENT = "virement"

class MouvementCreate(BaseModel):
    montant: float
    type_mouvement: TypeMouvementEnum
    mode_paiement: ModePaiementEnum
    description: str
    reference: str | None = None
    caisse_id: int

class MouvementResponse(BaseModel):
    id: int
    date: datetime
    montant: float
    type_mouvement: TypeMouvementEnum
    mode_paiement: ModePaiementEnum
    description: str
    reference: str | None
    
    class Config:
        from_attributes = True

class SoldeJournalierResponse(BaseModel):
    date: datetime
    solde_debut: float
    total_encaissements: float
    total_decaissements: float
    solde_fin: float
```

### `backend/api/routes/mouvements.py`
```python
from fastapi import APIRouter, Depends, HTTPException, status
from sqlalchemy.orm import Session
from typing import List
from ...models.database import get_db
from ...services.caisse_service import CaisseService
from ..schemas import MouvementCreate, MouvementResponse, SoldeJournalierResponse
from datetime import date

router = APIRouter(prefix="/mouvements", tags=["mouvements"])

@router.post("/encaissement", response_model=MouvementResponse)
def encaisser(
    mouvement: MouvementCreate,
    db: Session = Depends(get_db),
    # user: User = Depends(get_current_user)  # À décommenter avec auth
):
    """Enregistrer un encaissement"""
    service = CaisseService(db)
    
    if mouvement.type_mouvement != "encaissement":
        raise HTTPException(status_code=400, detail="Type de mouvement incorrect")
    
    try:
        result = service.encaisser(
            montant=mouvement.montant,
            mode=mouvement.mode_paiement,
            description=mouvement.description,
            caisse_id=mouvement.caisse_id,
            utilisateur_id=1,  # Remplacer par user.id
            reference=mouvement.reference
        )
        return result
    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e))

@router.post("/decaissement", response_model=MouvementResponse)
def decaisser(
    mouvement: MouvementCreate,
    db: Session = Depends(get_db)
):
    """Enregistrer un décaissement"""
    service = CaisseService(db)
    
    if mouvement.type_mouvement != "decaissement":
        raise HTTPException(status_code=400, detail="Type de mouvement incorrect")
    
    try:
        result = service.decaisser(
            montant=mouvement.montant,
            mode=mouvement.mode_paiement,
            description=mouvement.description,
            caisse_id=mouvement.caisse_id,
            utilisateur_id=1,
            reference=mouvement.reference
        )
        return result
    except ValueError as e:
        raise HTTPException(status_code=400, detail=str(e))

@router.get("/solde-journalier", response_model=SoldeJournalierResponse)
def get_solde_journalier(
    caisse_id: int,
    date_jour: date = None,
    db: Session = Depends(get_db)
):
    """Obtenir le solde et récap d'une journée"""
    service = CaisseService(db)
    result = service.get_solde_journalier(caisse_id, date_jour)
    return result

@router.get("/export/excel")
def export_excel(
    date_debut: date,
    date_fin: date,
    db: Session = Depends(get_db)
):
    """Exporter les mouvements vers Excel"""
    from ...services.export_service import ExportService
    
    mouvements = db.query(Mouvement).filter(
        Mouvement.date >= date_debut,
        Mouvement.date <= date_fin
    ).all()
    
    excel_data = ExportService.export_to_excel(mouvements)
    
    from fastapi.responses import StreamingResponse
    return StreamingResponse(
        io.BytesIO(excel_data),
        media_type="application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
        headers={"Content-Disposition": f"attachment; filename=mouvements_{date_debut}_{date_fin}.xlsx"}
    )
```

### `backend/api/main.py`
```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from .routes import mouvements, caisses, auth
from ..models.database import engine, Base

# Créer les tables
Base.metadata.create_all(bind=engine)

app = FastAPI(title="Gestion de Caisse API", version="1.0.0")

# CORS pour permettre au frontend web d'appeler l'API
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000", "http://localhost:5173"],  # URLs frontend
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Inclusion des routes
app.include_router(mouvements.router)
app.include_router(caisses.router)
app.include_router(auth.router)

@app.get("/")
def root():
    return {"message": "API Gestion de Caisse", "docs": "/docs"}

# Démarrer avec: uvicorn backend.api.main:app --reload
```

---

## 4️⃣ VERSION BUREAU (Tkinter)

### `desktop/main.py`
```python
import sys
import os
sys.path.append(os.path.dirname(os.path.dirname(os.path.abspath(__file__))))

import tkinter as tk
from tkinter import ttk, messagebox
from customtkinter import *
from backend.models.database import SessionLocal
from backend.services.caisse_service import CaisseService
from backend.models.mouvement import ModePaiement

# Configuration CustomTkinter
set_appearance_mode("dark")  # Modes: "System", "Dark", "Light"
set_default_color_theme("blue")

class CaisseApp(CTk):
    def __init__(self):
        super().__init__()
        
        self.title("Gestion de Caisse - Version Bureau")
        self.geometry("1200x700")
        
        # Connexion DB
        self.db = SessionLocal()
        self.service = CaisseService(self.db)
        
        # Widgets
        self.setup_ui()
        self.refresh_solde()
        
    def setup_ui(self):
        """Créer l'interface"""
        # Frame principale
        self.main_frame = CTkFrame(self)
        self.main_frame.pack(fill="both", expand=True, padx=20, pady=20)
        
        # Barre latérale gauche (actions)
        sidebar = CTkFrame(self.main_frame, width=250, corner_radius=10)
        sidebar.pack(side="left", fill="y", padx=(0, 10))
        sidebar.pack_propagate(False)
        
        CTkLabel(sidebar, text="ACTIONS", font=("Arial", 18, "bold")).pack(pady=20)
        
        CTkButton(sidebar, text="➕ Encaissement", command=self.open_encaissement,
                 height=50, font=("Arial", 14)).pack(pady=10, padx=20, fill="x")
        
        CTkButton(sidebar, text="➖ Décaissement", command=self.open_decaissement,
                 height=50, font=("Arial", 14)).pack(pady=10, padx=20, fill="x")
        
        CTkButton(sidebar, text="📊 Rapports", command=self.open_rapports,
                 height=50, font=("Arial", 14)).pack(pady=10, padx=20, fill="x")
        
        CTkButton(sidebar, text="📎 Export Excel", command=self.export_excel,
                 height=50, font=("Arial", 14)).pack(pady=10, padx=20, fill="x")
        
        # Zone principale droite
        right_area = CTkFrame(self.main_frame)
        right_area.pack(side="right", fill="both", expand=True)
        
        # Cadre solde
        solde_frame = CTkFrame(right_area, corner_radius=10)
        solde_frame.pack(fill="x", pady=(0, 20))
        
        self.solde_label = CTkLabel(solde_frame, text="Solde actuel : 0.00 €",
                                   font=("Arial", 32, "bold"), text_color="green")
        self.solde_label.pack(pady=30)
        
        # Tableau des derniers mouvements
        CTkLabel(right_area, text="DERNIERS MOUVEMENTS", font=("Arial", 16, "bold")).pack(anchor="w")
        
        # Treeview pour tableau
        self.tree = ttk.Treeview(right_area, columns=("Date", "Type", "Montant", "Mode", "Description"), 
                                 show="headings", height=15)
        
        self.tree.heading("Date", text="Date")
        self.tree.heading("Type", text="Type")
        self.tree.heading("Montant", text="Montant")
        self.tree.heading("Mode", text="Mode")
        self.tree.heading("Description", text="Description")
        
        self.tree.column("Date", width=150)
        self.tree.column("Type", width=100)
        self.tree.column("Montant", width=100)
        self.tree.column("Mode", width=120)
        self.tree.column("Description", width=300)
        
        self.tree.pack(fill="both", expand=True, pady=10)
        
        # Scrollbar
        scrollbar = ttk.Scrollbar(right_area, orient="vertical", command=self.tree.yview)
        scrollbar.pack(side="right", fill="y")
        self.tree.configure(yscrollcommand=scrollbar.set)
        
    def refresh_solde(self):
        """Actualiser l'affichage du solde et du tableau"""
        from backend.models.caisse import Caisse
        
        caisse = self.db.query(Caisse).first()
        if caisse:
            solde = caisse.solde_actuel
            self.solde_label.configure(text=f"Solde actuel : {solde:.2f} €")
            
            # Rafraîchir tableau
            for item in self.tree.get_children():
                self.tree.delete(item)
            
            # Afficher les 20 derniers mouvements
            mouvements = self.db.query(Mouvement).order_by(Mouvement.date.desc()).limit(20).all()
            for m in mouvements:
                self.tree.insert("", "end", values=(
                    m.date.strftime("%d/%m/%Y %H:%M"),
                    "↑ ENCAISSEMENT" if m.type_mouvement.value == "encaissement" else "↓ DECAISSEMENT",
                    f"{m.montant:.2f} €",
                    m.mode_paiement.value,
                    m.description[:50]
                ))
        
        self.after(5000, self.refresh_solde)  # Rafraîchir toutes les 5s
    
    def open_encaissement(self):
        """Fenêtre d'encaissement"""
        dialog = CTkToplevel(self)
        dialog.title("Encaissement")
        dialog.geometry("500x500")
        dialog.grab_set()
        
        CTkLabel(dialog, text="NOUVEL ENCAISSEMENT", font=("Arial", 20, "bold")).pack(pady=20)
        
        # Montant
        CTkLabel(dialog, text="Montant (€):").pack(pady=(20, 5))
        montant_entry = CTkEntry(dialog, placeholder_text="0.00")
        montant_entry.pack(pady=5)
        
        # Mode de paiement
        CTkLabel(dialog, text="Mode de paiement:").pack(pady=(10, 5))
        mode_var = tk.StringVar(value="espèces")
        mode_menu = CTkOptionMenu(dialog, values=[m.value for m in ModePaiement], variable=mode_var)
        mode_menu.pack(pady=5)
        
        # Description
        CTkLabel(dialog, text="Description:").pack(pady=(10, 5))
        desc_text = CTkTextbox(dialog, height=100)
        desc_text.pack(pady=5, padx=20, fill="x")
        
        # Référence
        CTkLabel(dialog, text="Référence (facture, etc.):").pack(pady=(10, 5))
        ref_entry = CTkEntry(dialog)
        ref_entry.pack(pady=5)
        
        def valider():
            try:
                montant = float(montant_entry.get())
                mode = mode_var.get()
                description = desc_text.get("1.0", "end-1c")
                reference = ref_entry.get() or None
                
                # Récupérer caisse par défaut
                from backend.models.caisse import Caisse
                caisse = self.db.query(Caisse).first()
                
                mouvement = self.service.encaisser(
                    montant=montant,
                    mode=ModePaiement(mode),
                    description=description,
                    caisse_id=caisse.id,
                    utilisateur_id=1,  # À gérer avec auth
                    reference=reference
                )
                
                messagebox.showinfo("Succès", f"Encaissement de {montant}€ enregistré!")
                dialog.destroy()
                self.refresh_solde()
                
            except ValueError as e:
                messagebox.showerror("Erreur", str(e))
        
        CTkButton(dialog, text="VALIDER L'ENCAISSEMENT", command=valider,
                 fg_color="green", height=40).pack(pady=30)
    
    def open_decaissement(self):
        """Fenêtre de décaissement (similaire)"""
        # Structure identique à encaissement mais appelle service.decaisser()
        pass
    
    def export_excel(self):
        """Exporter vers Excel"""
        from backend.services.export_service import ExportService
        from tkinter import filedialog
        
        filename = filedialog.asksaveasfilename(
            defaultextension=".xlsx",
            filetypes=[("Excel files", "*.xlsx"), ("All files", "*.*")]
        )
        
        if filename:
            mouvements = self.db.query(Mouvement).all()
            excel_data = ExportService.export_to_excel(mouvements)
            
            with open(filename, "wb") as f:
                f.write(excel_data)
            
            messagebox.showinfo("Export", f"Exporté vers {filename}")
    
    def on_closing(self):
        """Fermer la connexion DB"""
        self.db.close()
        self.destroy()

if __name__ == "__main__":
    # Créer une caisse par défaut si aucune n'existe
    from backend.models.database import Base, engine
    from backend.models.caisse import Caisse
    
    Base.metadata.create_all(bind=engine)
    
    db = SessionLocal()
    if not db.query(Caisse).first():
        caisse_defaut = Caisse(nom="Caisse principale", solde_initial=1000)
        db.add(caisse_defaut)
        db.commit()
    db.close()
    
    app = CaisseApp()
    app.protocol("WM_DELETE_WINDOW", app.on_closing)
    app.mainloop()
```

---

## 5️⃣ FICHIER REQUIREMENTS

```txt
# Core
sqlalchemy==2.0.23
pydantic==2.4.2

# API (version web)
fastapi==0.104.1
uvicorn[standard]==0.24.0
python-multipart==0.0.6
python-jose[cryptography]==3.3.0
passlib[bcrypt]==1.7.4

# Bureau
customtkinter==5.2.1
Pillow==10.1.0

# Exports
pandas==2.1.2
openpyxl==3.1.2
reportlab==4.0.4

# Utilitaires
python-dotenv==1.0.0
```

---

## 6️⃣ COMMENT LANCER CHAQUE VERSION

### Version WEB
```bash
# Terminal 1 - Backend API
cd gestion_caisse
uvicorn backend.api.main:app --reload --port 8000
# Accès API docs: http://localhost:8000/docs

# Terminal 2 - Frontend React (à créer séparément)
cd web/frontend
npm run dev  # Port 5173 par défaut
```

### Version BUREAU
```bash
cd gestion_caisse
python desktop/main.py
# Ou créer un .exe avec PyInstaller:
pyinstaller --onefile --windowed --name "GestionCaisse" desktop/main.py
```

---

## 7️⃣ POINTS CLÉS DE CETTE ARCHITECTURE

| Avantage | Explication |
|----------|-------------|
| **Code métier unique** | Les services (`CaisseService`) sont utilisés par les deux versions |
| **Base unique** | SQLite pour bureau (fichier local), PostgreSQL pour web |
| **Flexibilité** | La version bureau peut aussi appeler l'API (mode connecté) |
| **Maintenabilité** | Ajouter une fonction = disponible partout |
| **Sécurité** | API avec JWT pour le web, accès direct DB pour bureau local |

---

## 8️⃣ PROCHAINES ÉTAPES POUR ALLER PLUS LOIN

1. **Ajouter l'authentification** (JWT pour web, login simple pour bureau)
2. **Multi-caisses** (plusieurs comptes de caisse)
3. **Dashboard avec graphiques** (Matplotlib pour bureau, Chart.js pour web)
4. **Backup automatique** des données SQLite
5. **Mode déconnecté** pour la version bureau (sync plus tard)

Tu veux que je détaille un de ces aspects ou que je crée le frontend React pour la version web ?
