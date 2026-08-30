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
