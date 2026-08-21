# Transcrição de MKV com separação de falantes

Pipeline 100% local: extrai áudio dos `.mkv`, separa quem fala (diarização),
transcreve (Whisper) e gera um `.docx` por vídeo. Roda na **CPU** — a GPU AMD
não é usada nesta etapa (o suporte a GPU AMD no Windows para esses modelos é
instável; CPU é o caminho garantido). Se depois você quiser usar a GPU, veja
a seção "Usando a GPU AMD depois" no final.

## 1. Instalar pré-requisitos

**Python 3.10 ou 3.11** — baixe em https://python.org (marque "Add to PATH" no instalador).

**ffmpeg** — abra o PowerShell e rode:
```powershell
winget install ffmpeg
```
Feche e reabra o PowerShell depois, para o PATH atualizar.

## 2. Criar ambiente virtual e instalar dependências

No PowerShell, dentro da pasta onde estão os arquivos `.py`:
```powershell
python -m venv venv
venv\Scripts\activate

pip install torch --index-url https://download.pytorch.org/whl/cpu
pip install -r requirements.txt
```

## 3. Criar um token do Hugging Face (necessário para a diarização)

O modelo de diarização (`pyannote/speaker-diarization-3.1`) é "gated" — você
precisa aceitar os termos de uso antes de conseguir baixá-lo:

1. Crie uma conta em https://huggingface.co
2. Acesse e clique em "Agree and access repository" nestas duas páginas:
   - https://huggingface.co/pyannote/speaker-diarization-3.1
   - https://huggingface.co/pyannote/segmentation-3.0
3. Crie um token de acesso em https://huggingface.co/settings/tokens (tipo "Read" é suficiente)
4. Defina o token como variável de ambiente (evita digitar toda vez):
```powershell
setx HF_TOKEN "seu_token_aqui"
```
   (feche e reabra o PowerShell depois de rodar isso)

## 4. Rodar

Um único vídeo:
```powershell
python transcribe_diarize.py --input "C:\Videos\reuniao.mkv" --output saida
```

Uma pasta inteira de vídeos `.mkv`:
```powershell
python transcribe_diarize.py --input "C:\Videos" --output saida
```

Cada vídeo gera um `.docx` correspondente na pasta `saida`, com trechos
rotulados `[00:01:23] Pessoa 1: ...`, `[00:01:45] Pessoa 2: ...` etc.

### Opções úteis

| Opção | Padrão | Descrição |
|---|---|---|
| `--model` | `medium` | Tamanho do Whisper: `tiny`, `base`, `small`, `medium`, `large-v3`. Maior = mais lento e mais preciso. Em CPU, `small` ou `medium` costuma ser o melhor equilíbrio. |
| `--language` | `pt` | Idioma do áudio. Use `auto` se os vídeos tiverem idiomas variados. |

### Sobre o tempo de processamento

Em CPU, a transcrição costuma levar entre 0,5x e 2x a duração do vídeo
(dependendo do tamanho do modelo e do seu processador), e a diarização
adiciona um tempo parecido. Um vídeo de 1 hora pode levar de 1 a 3 horas
no total com o modelo `medium`. Para testar mais rápido, use `--model small`
primeiro.

## 5. (Opcional) Revisar o texto com um LLM local usando a GPU AMD

O script `polish_with_local_llm.py` pega os `.docx` já gerados e usa um LLM
local (via [Ollama](https://ollama.com/download)) para corrigir pontuação e
limpar vícios de fala, mantendo o conteúdo. Essa é a parte onde sua Radeon
XTX de 20GB entra em ação:

1. Instale o Ollama para Windows.
2. Baixe um modelo, ex.: `ollama pull llama3.1:8b` (ou um modelo maior, já
   que você tem 20GB de VRAM — ex. `ollama pull qwen2.5:14b`).
3. Rode:
```powershell
python polish_with_local_llm.py --input saida --output saida_revisada
```

O Ollama detecta e usa GPUs AMD compatíveis automaticamente no Windows
(via ROCm) — não é preciso configurar nada manualmente na maioria dos casos.
Verifique a lista de GPUs suportadas na documentação do Ollama se o modelo
não usar a GPU.

## Usando a GPU AMD depois (transcrição/diarização mais rápidas)

Se no futuro quiser acelerar a transcrição e a diarização em si (não só o
LLM), as opções mais viáveis para AMD no Windows são:
- **WSL2 + ROCm**: melhor desempenho, mas exige rodar dentro do subsistema
  Linux do Windows.
- **whisper.cpp com backend Vulkan**: alternativa ao faster-whisper que
  roda em GPUs AMD via Vulkan, sem precisar de ROCm.

Ambas exigem trocar a peça de transcrição do pipeline — me avise se quiser
que eu adapte o script para uma dessas rotas mais tarde.

## Solução de problemas

- **`ffmpeg` não encontrado**: reabra o PowerShell depois de instalar, ou
  reinicie o PC.
- **Erro 401/403 ao baixar o modelo de diarização**: confirme que aceitou os
  termos nas duas páginas do Hugging Face listadas acima, com a mesma conta
  do token.
- **Muito lento**: use `--model small` ou `--model base` para testar antes
  de rodar com `medium`/`large-v3` em vídeos longos.
