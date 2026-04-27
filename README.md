#  interface PYQT5

import sys
from PyQt5.QtWidgets import (
    QApplication, QMainWindow, QWidget, QVBoxLayout, QHBoxLayout,
    QPushButton, QTextEdit, QFileDialog, QMessageBox
)
from PyQt5.QtGui import QFont
from PyQt5.QtCore import Qt

# Définition class arbre et noeud ainsi que les fonctions recherche
import csv
import random

class Noeud:
    def __init__(self, prix):
        self.prix = prix
        self.joueurs = []
        self.gauche = None
        self.droite = None

class ABR:
    def __init__(self):
        self.racine = None

    def inserer(self, prix, joueur):
        self.racine = self._inserer_rec(self.racine, prix, joueur)

    def _inserer_rec(self, noeud, prix, joueur):
        if noeud is None:
            n = Noeud(prix)
            n.joueurs.append(joueur)
            return n

        if prix == noeud.prix:
            noeud.joueurs.append(joueur)
        elif prix < noeud.prix:
            noeud.gauche = self._inserer_rec(noeud.gauche, prix, joueur)
        else:
            noeud.droite = self._inserer_rec(noeud.droite, prix, joueur)

        return noeud

    def parcours_infixe_liste(self):
        result = []
        self._infixe_liste(self.racine, result)
        return result

    def _infixe_liste(self, noeud, result):
        if noeud:
            self._infixe_liste(noeud.gauche, result)
            result.append(f"Prix {noeud.prix} -> joueurs: {noeud.joueurs}")
            self._infixe_liste(noeud.droite, result)

    def plus_bas_unique(self):
        return self._plus_bas_unique(self.racine)

    def _plus_bas_unique(self, noeud):
        if noeud is None:
            return None

        gauche = self._plus_bas_unique(noeud.gauche)
        if gauche:
            return gauche

        if len(noeud.joueurs) == 1:
            return (noeud.prix, noeud.joueurs[0])

        return self._plus_bas_unique(noeud.droite)

# Calcul pour la simulation

def calcul_recette(abr, cout_base, alpha):
    total = 0

    def parcours(noeud):
        nonlocal total
        if noeud:
            parcours(noeud.gauche)
            for _ in noeud.joueurs:
                cout = cout_base + alpha / (noeud.prix + 1)
                total += cout
            parcours(noeud.droite)

    parcours(abr.racine)
    return total

# simulation

def joueur_aleatoire():
    return random.randint(0, 50)

def joueur_strategique():
    return random.randint(5, 30)

def simulation(nb_manches=500, nb_joueurs=20):
    victoires = {"aleatoire": 0, "strategique": 0}
    gains = {"aleatoire": 0, "strategique": 0}
    recettes = []

    for _ in range(nb_manches):
        abr = ABR()
        joueurs = []

        for i in range(nb_joueurs):
            if i < nb_joueurs // 2:
                prix = joueur_aleatoire()
                joueurs.append(("aleatoire", prix))
            else:
                prix = joueur_strategique()
                joueurs.append(("strategique", prix))

        for j, prix in joueurs:
            abr.inserer(prix, j)

        gagnant = abr.plus_bas_unique()

        if gagnant:
            prix, type_joueur = gagnant
            victoires[type_joueur] += 1
            gains[type_joueur] += prix

        recette = calcul_recette(abr, 1, 10)
        recettes.append(recette)

    return victoires, gains, recettes

# interface PYQT5

class MainWindow(QMainWindow):
    def __init__(self):
        super().__init__()

        self.setWindowTitle("LowBid Pro")
        self.setGeometry(200, 200, 900, 600)

        self.abr = ABR()

        # Layout principal
        central = QWidget()
        self.setCentralWidget(central)
        layout = QVBoxLayout()

        # Boutons
        btn_layout = QHBoxLayout()

        self.btn_csv = QPushButton("Charger CSV")
        self.btn_sim = QPushButton("Simulation")
        self.btn_reset = QPushButton("Reset")

        for btn in [self.btn_csv, self.btn_sim, self.btn_reset]:
            btn.setFixedHeight(40)
            btn.setCursor(Qt.PointingHandCursor)

        btn_layout.addWidget(self.btn_csv)
        btn_layout.addWidget(self.btn_sim)
        btn_layout.addWidget(self.btn_reset)

        # Zone texte
        self.text = QTextEdit()
        self.text.setFont(QFont("Consolas", 10))
        self.text.setReadOnly(True)


        layout.addLayout(btn_layout)
        layout.addWidget(self.text)
        central.setLayout(layout)

        # style et couleur de l'interface blanc / bleu
        self.setStyleSheet("""
            QMainWindow {
                background-color: white;
            }

            QPushButton {
                background-color: #1E88E5;
                color: white;
                border-radius: 8px;
                font-weight: bold;
            }

            QPushButton:hover {
                background-color: #1565C0;
            }

            QTextEdit {
                background-color: #F5F7FA;
                border: 1px solid #D0D7E2;
                padding: 10px;
            }
        """)

        # connexion
        self.btn_csv.clicked.connect(self.charger_csv)
        self.btn_sim.clicked.connect(self.lancer_simulation)
        self.btn_reset.clicked.connect(self.reset)

    # action de charger les fichiers

    def charger_csv(self):
        self.abr = ABR()

        fichier, _ = QFileDialog.getOpenFileName(self, "Choisir un CSV", "", "CSV (*.csv)")
        if not fichier:
            return

        try:
            with open(fichier, newline='', encoding='utf-8') as f:
                reader = csv.reader(f)
                next(reader)

                for ligne in reader:
                    try:
                        joueur = ligne[0]
                        prix = int(ligne[1])
                    except:
                        joueur = ligne[1]
                        prix = int(ligne[2])

                    self.abr.inserer(prix, joueur)

            self.afficher_resultats()

        except Exception as e:
            QMessageBox.critical(self, "Erreur", str(e))

    def afficher_resultats(self):
        self.text.clear()

        self.text.append("=== ETAT DE L'ENCHERE ===\n")
        for ligne in self.abr.parcours_infixe_liste():
            self.text.append(ligne)

        gagnant = self.abr.plus_bas_unique()

        self.text.append("\n=== RESULTAT ===")
        if gagnant:
            self.text.append(f"Gagnant : prix = {gagnant[0]}, joueur = {gagnant[1]}")
        else:
            self.text.append("Aucun gagnant")

        recette = calcul_recette(self.abr, 1, 10)
        self.text.append(f"Recette vendeur : {recette:.2f}")

    def lancer_simulation(self):
        victoires, gains, recettes = simulation()

        self.text.clear()
        self.text.append("=== SIMULATION COMPLETE ===\n")

        total_victoires = sum(victoires.values())
        for k in victoires:
            taux = (victoires[k] / total_victoires * 100) if total_victoires > 0 else 0
            self.text.append(f"{k} : {victoires[k]} ({taux:.2f}%)")

        self.text.append("\nGain moyen :")
        for k in gains:
            gm = gains[k] / victoires[k] if victoires[k] > 0 else 0
            self.text.append(f"{k} : {gm:.2f}")

        recette_moy = sum(recettes) / len(recettes)
        self.text.append(f"\nRecette moyenne vendeur : {recette_moy:.2f}")

    def reset(self):
        self.text.clear()

# lancement
if __name__ == "__main__":
    app = QApplication(sys.argv)
    window = MainWindow()
    window.show()
    sys.exit(app.exec_())
