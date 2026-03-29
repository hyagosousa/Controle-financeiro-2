<!DOCTYPE html>
<html lang="pt-BR">
<head>
  <meta charset="UTF-8">
  <title>Analisador de Balancete</title>

  <script src="https://cdnjs.cloudflare.com/ajax/libs/pdf.js/2.16.105/pdf.min.js"></script>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/jszip/3.10.1/jszip.min.js"></script>

  <style>
    body {
      font-family: Arial;
      background: #111;
      color: white;
      text-align: center;
      padding: 20px;
    }

    h1 { color: #00ffcc; }

    input { margin: 20px; padding: 10px; }

    button {
      padding: 10px 20px;
      margin: 10px;
      background: #00aa88;
      border: none;
      border-radius: 5px;
      color: white;
      cursor: pointer;
    }

    .box {
      display: flex;
      justify-content: space-around;
      margin-top: 20px;
    }

    .coluna {
      width: 45%;
      padding: 15px;
      border-radius: 10px;
    }

    .positivo { background: #063; }
    .negativo { background: #600; }

    li { margin: 8px 0; text-align: left; }

    .analise {
      margin-top: 30px;
      background: #222;
      padding: 15px;
      border-radius: 10px;
      text-align: left;
    }
  </style>
</head>

<body>

<h1>📄 Analisador de Balancete</h1>

<input type="file" id="pdfInput" multiple accept="application/pdf">
<br>
<button onclick="baixarNegativosZIP()">⬇️ Baixar Negativos (ZIP)</button>

<div class="box">
  <div class="coluna positivo">
    <h2>✅ Positivos</h2>
    <ul id="positivos"></ul>
  </div>

  <div class="coluna negativo">
    <h2>❌ Negativos</h2>
    <ul id="negativos"></ul>
  </div>
</div>

<div class="analise">
  <h2>📊 Análise Avançada</h2>
  <div id="resultadoAnalise"></div>
</div>

<script>
const input = document.getElementById("pdfInput");
let arquivosNegativos = [];

input.addEventListener("change", async (event) => {

  document.getElementById("positivos").innerHTML = "";
  document.getElementById("negativos").innerHTML = "";
  document.getElementById("resultadoAnalise").innerHTML = "";
  arquivosNegativos = [];

  const arquivos = event.target.files;

  for (let file of arquivos) {
    const texto = await lerPDF(file);
    analisarTexto(texto, file.name, file);
    calcularAnaliseAvancada(texto, file.name);
  }
});

async function lerPDF(file) {
  const reader = new FileReader();

  return new Promise((resolve) => {
    reader.onload = async function () {
      const typedarray = new Uint8Array(this.result);
      const pdf = await pdfjsLib.getDocument(typedarray).promise;

      let texto = "";

      for (let i = 1; i <= pdf.numPages; i++) {
        const pagina = await pdf.getPage(i);
        const conteudo = await pagina.getTextContent();

        conteudo.items.forEach(item => {
          texto += item.str + " ";
        });

        texto += "\n"; // mantém quebra de linha
      }

      resolve(texto.toLowerCase());
    };

    reader.readAsArrayBuffer(file);
  });
}

// 🔢 converte número BR
function extrairNumero(valor) {
  valor = valor.replace(/\./g, "").replace(",", ".");
  return parseFloat(valor.replace(/[^\d.-]/g, "")) || 0;
}

// 🔍 CLASSIFICAÇÃO
function analisarTexto(texto, nomeArquivo, fileAtual) {

  const linhas = texto.split("\n");

  for (let linha of linhas) {

    if (linha.includes("resultado do período")) {

      const numeros = linha.match(/\(?\s*\d{1,3}(?:\.\d{3})*,\d{2}\s*\)?/g);

      if (numeros && numeros.length >= 4) {

        const saldoBruto = numeros[3];
        const saldo = saldoBruto.replace(/\s+/g, "");

        const ehNegativo =
          saldo.startsWith("(") ||
          saldo.endsWith(")");

        if (ehNegativo) {
          adicionarLista("negativos", nomeArquivo, saldo);
          arquivosNegativos.push(fileAtual);
        } else {
          adicionarLista("positivos", nomeArquivo, saldo);
        }

        break;
      }
    }
  }
}

function adicionarLista(tipo, nome, saldo) {
  const ul = document.getElementById(tipo);
  const li = document.createElement("li");
  li.textContent = `${nome} → Saldo: ${saldo}`;
  ul.appendChild(li);
}

// 📊 ANÁLISE AVANÇADA (BLINDADA)
function calcularAnaliseAvancada(texto, nomeArquivo) {

  const linhas = texto.split("\n");

  let valorServico = 0;
  let valorSimples = 0;
  let resultadoPeriodo = 0;

  for (let linha of linhas) {

    // PRESTAÇÃO DE SERVIÇOS
    if (linha.includes("prestação de serviços")) {

      const numeros = linha.match(/\d{1,3}(\.\d{3})*,\d{2}/g);

      if (numeros && numeros.length > 0) {
        valorServico = extrairNumero(numeros[numeros.length - 1]);
      }
    }

    // SIMPLES NACIONAL
    if (linha.includes("simples nacional")) {

      const numeros = linha.match(/\(?\s*\d{1,3}(?:\.\d{3})*,\d{2}\s*\)?/g);

      if (numeros && numeros.length > 0) {
        valorSimples = extrairNumero(numeros[numeros.length - 1]);
      }
    }

    // RESULTADO DO PERÍODO
    if (linha.includes("resultado do período")) {

      const numeros = linha.match(/\(?\s*\d{1,3}(?:\.\d{3})*,\d{2}\s*\)?/g);

      if (numeros && numeros.length >= 4) {
        resultadoPeriodo = extrairNumero(numeros[3]);
      }
    }
  }

  const calc1 = valorServico * 0.32;
  const calc2 = valorSimples * 0.05;
  const resultadoFinal = calc1 - calc2;

  const status = resultadoFinal > resultadoPeriodo ? "✅ MAIOR" : "❌ MENOR";

  const div = document.getElementById("resultadoAnalise");

  div.innerHTML += `
    <p>
      <strong>${nomeArquivo}</strong><br>
      Serviço: R$ ${valorServico.toLocaleString('pt-BR')}<br>
      Simples: R$ ${valorSimples.toLocaleString('pt-BR')}<br>
      Resultado Calc: R$ ${resultadoFinal.toLocaleString('pt-BR')}<br>
      Resultado Período: R$ ${resultadoPeriodo.toLocaleString('pt-BR')}<br>
      <strong>${status}</strong>
    </p>
    <hr>
  `;
}

// 📦 ZIP
async function baixarNegativosZIP() {

  if (arquivosNegativos.length === 0) {
    alert("Nenhum PDF negativo encontrado.");
    return;
  }

  const zip = new JSZip();

  arquivosNegativos.forEach(file => {
    zip.file(file.name, file);
  });

  const conteudo = await zip.generateAsync({ type: "blob" });

  const link = document.createElement("a");
  link.href = URL.createObjectURL(conteudo);
  link.download = "pdfs_negativos.zip";
  link.click();
}
</script>

</body>
</html>
