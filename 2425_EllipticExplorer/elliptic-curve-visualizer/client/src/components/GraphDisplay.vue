<template>
  <div id="desmos-view" class="desmos-container"></div>
</template>

<script setup>
import { onMounted, watch } from "vue";
import { useCurveStore } from "@/stores/curveStore";
import Desmos from "desmos";
import { defineEmits } from "vue";

const store = useCurveStore();
let calculator = null;
let selectedPoint = null;

// Définition de l'événement pour la sélection de point
const emit = defineEmits(["pointSelected"]);

onMounted(() => {
  const elt = document.getElementById("desmos-view");
  calculator = Desmos.GraphingCalculator(elt, {
    expressions: false,
    settingsMenu: false,
    zoomButtons: true,
    invertedColors: false,
    xAxisStep: 1,
    yAxisStep: 1,
    xAxisLabel: 'x',
    yAxisLabel: 'y',
    restrictedFunctions: false
  });

  // Ajouter un gestionnaire d'événements pour le clic sur le graphique
  elt.addEventListener("click", handleGraphClick);

  renderCurve();
});

watch(
  () => [store.curveType, store.field, store.graph2D],
  () => {
    renderCurve();
  },
  { deep: true }
);

function renderCurve() {
  if (!calculator) return;
  calculator.setBlank();

  if (store.field === "real") {
    renderImplicitEquation();
  } else if (store.field === "modulo") {
    renderPointCloud();
  }
}

function renderImplicitEquation() {
  let latex = "";
  const a = store.a ?? 1;
  const b = store.b ?? 1;
  const d = store.d ?? 1;

  switch (store.curveType) {
    case "weierstrass":
      latex = `y^2 = x^3 + ${a}x + ${b}`;
      break;
    case "montgomery":
      latex = `y^2 = x^3 + ${a}x^2 + ${b}x`;
      break;
    case "edwards":
      latex = `x^2 + y^2 = 1 + ${d}x^2y^2`;
      break;
  }

  if (latex) {
    calculator.setExpression({
      id: "curve",
      latex,
      color: Desmos.Colors.BLUE,
      lineStyle: Desmos.Styles.SOLID
    });
  }
}

function renderPointCloud() {
  const data = store.graph2D;
  if (!data || data.length === 0) return;

  const exprs = data.map((pt, i) => ({
    id: `pt${i}`,
    latex: `(${pt.x}, ${pt.y})`,
    color: Desmos.Colors.RED,
    pointStyle: Desmos.Styles.POINT
  }));

  exprs.forEach(e => calculator.setExpression(e));
}

// Fonction pour gérer le clic sur le graphique
function handleGraphClick(event) {
  if (!calculator) return;

  // Obtenir les coordonnées du clic
  const rect = event.target.getBoundingClientRect();
  const x = calculator.pixelsToMath({ x: event.clientX - rect.left, y: 0 }).x;
  const y = calculator.pixelsToMath({ x: 0, y: event.clientY - rect.top }).y;

  // Émettre l'événement de sélection de point
  emit("pointSelected", { x, y });

  // Ajouter le point sélectionné au graphique
  if (selectedPoint) {
    calculator.removeExpression({ id: "selectedPoint" }); // Supprimer l'ancien point
  }
  selectedPoint = { x, y };
  calculator.setExpression({
    id: "selectedPoint",
    latex: `(${x}, ${y})`,
    color: Desmos.Colors.GREEN, // Couleur jaune
    pointStyle: Desmos.Styles.POINT,
    size: 10, // Taille plus grande
  });
}
</script>

<style scoped>
.desmos-container {
  width: 100%;
  height: 100%;
  border: 1px solid #ccc;
}
</style>
