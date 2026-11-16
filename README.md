# El Bobo — Jogo 2D em Unity

**El Bobo** é um jogo de plataforma 2D desenvolvido em **Unity (C#)**, com foco em mecânicas de combate, IA de inimigos, sistema de vida, menu de pausa/configurações e transições de cena.

---

## Visão Geral

| Aspecto | Detalhes |
|---------|----------|
| **Engine** | Unity (2D) |
| **Linguagem** | C# (.NET) |
| **Gênero** | Plataforma / Ação / Aventura |
| **Perspectiva** | Side-scroller 2D |
| **Sistema de Build** | Unity Package Manager (Packages/) |

---

## Estrutura do Projeto

```
ElBobo/
├── Assets/
│   ├── Scripts/                    # Código principal (C#)
│   │   ├── Jogador.cs              # Player base (parcial)
│   │   ├── SimpleMove.cs           # Movimento, pulo, vida, knockback, interação
│   │   ├── BoboAttack.cs           # Sistema de ataque (espada/cetro/arco)
│   │   ├── vida.cs                 # Coleta de vida (pickup)
│   │   ├── camera.cs               # Camera follow com limites
│   │   ├── InimigoIA.cs            # IA base: patrol, detecção borda/parede, perseguição
│   │   ├── InimigoSoldado_espada.cs # Inimigo melee que persegue player
│   │   ├── InimigoArqueiro.cs      # Inimigo ranged (arqueiro)
│   │   ├── Dragon.cs               # Boss dragão
│   │   ├── LifeEnemy.cs            # Sistema de vida dos inimigos + barra de HP
│   │   ├── KnockBack.cs            # Empurrão ao receber dano
│   │   ├── HitStop.cs              # Hit pause (screen shake temporal)
│   │   ├── Menu.cs                 # Pause, opções (volume, qualidade, resolução, fullscreen)
│   │   ├── MainMenu.cs             # Menu principal
│   │   ├── MudarCena.cs            # Transição de cenas
│   │   ├── Spawner.cs              # Spawn de inimigos
│   │   ├── PlataformMovement.cs    # Plataformas móveis
│   │   ├── InteragirComObjetos.cs  # Interação com objetos (tecla F)
│   │   ├── Altar.cs                # Altar/mecânica especial
│   │   ├── Flecha.cs               # Projétil do arqueiro
│   │   ├── colidiu.cs              # Detecção de colisão genérica
│   │   ├── danoObjetos.cs          # Dano em objetos
│   │   ├── transicao.cs            # Transições visuais
│   │   ├── UIManagerScript.cs      # UI de vida (corações)
│   │   └── Save.cs                 # Sistema de save
│   ├── Scenes/                     # Cenas do jogo
│   ├── Prefabs/ / Prefab/          # Prefabs reutilizáveis
│   ├── Sprite Sheet/               # Spritesheets do personagem/inimigos
│   ├── PixelArtSteampunk/          # Assets de pixel art (estilo steampunk)
│   ├── Soundtrack/                 # Músicas e SFX
│   ├── Dialogue/                   # Sistema de diálogos
│   ├── Examples/                   # Exemplos/referências
│   └── superscript/                # Scripts auxiliares
├── ProjectSettings/                # Configurações do Unity
├── Packages/                       # Dependências UPM
└── .gitignore                      # Unity standard gitignore
```

---

## Principais Sistemas

### 1. Movimento do Jogador (`SimpleMove.cs`)
- **Movimento horizontal**: Input `Horizontal` (setas/WASD), velocidade configurável
- **Pulo**: `Jump` (espaço) com verificação `GroundCheck` via `Linecast` (layers `ground` e `plataforma`)
- **Flip automático**: Inverte `localScale.x` conforme direção
- **Plataformas móveis**: `OnCollisionEnter2D`/`Exit2D` parentiza player à plataforma
- **Morte por queda**: Detecta layer `ForaDeCena` → `TakeDamage(1)` + teleport para último chão válido
- **Interação**: Tecla `F` seta `EstaInteragindo = true`

### 2. Combate do Jogador (`BoboAttack.cs`)
- **Cooldown de ataque**: `attackCooldown` + timer
- **Tipos de arma** (booleanos mutuamente exclusivos):
  - `espada` → trigger `AtaqueEspada`
  - `cetro` → (planejado) `AtaqueCetro`
  - `arco` → (planejado) `AtaqueArco`
- **Detecção**: `Physics2D.OverlapCircleAll` no `attackPoint` com `attackRange` e `enemyLayers`
- **Dano**: Chama `LifeEnemy.TakeDamage(attackDamage)` em todos os alvos no alcance
- **Pause**: Respeita `Menu.isPaused`

### 3. Sistema de Vida do Jogador (`SimpleMove.cs` + `UIManagerScript.cs`)
- `health` (int) + `recoveryTime` (invencibilidade temporária)
- `TakeDamage(int)` → reduz vida, atualiza UI, inicia cooldown `recovering`
- `Die()` → desativa GameObject, `Time.timeScale = 0`, abre `deathOptions`
- **Knockback**: Corrotina `Knockback(duration, power, source)` aplica força oposta
- **UI**: `UIManagerScript.updateLives(health)` atualiza corações na tela

### 4. Coleta de Vida (`vida.cs`)
- `OnTriggerEnter2D` com tag `Player`
- Cura +1 HP se `health < 10`
- Atualiza UI e auto-destrói

### 5. Câmera (`camera.cs`)
- `Follow` com `Mathf.Clamp` nos eixos X e Y
- Limites configuráveis: `minimoX`, `maximoX`, `minimoY`, `maximoY`
- Suave (atualiza todo frame no `Update`)

### 6. IA de Inimigos

#### Base: `InimigoIA.cs`
- **Patrol**: Anda horizontalmente (`velocidadeHorizontal`) invertendo direção em paredes/bordas
- **Raycasts**:
  - Parede: `Raycast` lateral (0.3f) → inverte ou pula
  - Borda: `Raycast` para baixo (1f) → inverte se não há chão
- **Detecção Player**: `OverlapCircle` / distância X < `rangeDetectar` → `seguindoPlayer = true`
- **Modo Zumbi** (`modoZumbi`): Persegue player horizontalmente; pula ao encontrar parede
- **Stun**: `set_stun(true)` zera velocidade
- **Plataformas**: Parentiza igual ao player
- **Knockback/HitStop**: Referencia componentes do player no `Start()`

#### Soldado Espada (`InimigoSoldado_espada.cs`)
- Persegue player se distância > `distancia`
- `Vector2.MoveTowards` + flip via `eulerAngles`
- Colisão com player → `AddForce` horizontal (empurrão)

#### Arqueiro (`InimigoArqueiro.cs`) — *inferido*
- Dispara `Flecha.cs` (projétil)

#### Dragão / Boss (`Dragon.cs`)
- Static `ataque` property → trigger `Ataque` no Animator
- `GameOver()` → ativa menu de morte, dano massivo no player, instancia `deathOptions`

### 7. Sistema de Vida dos Inimigos (`LifeEnemy.cs`)
- `health` + `recoveryTime` (invencibilidade)
- **Barra de HP visual**: `healthBar` (Transform) escala X proporcional à vida
- `TakeDamage(int)` → reduz vida, atualiza barra, respeita `recovering`
- Morte: Trigger `Morrendo` no Animator → `Destroy(inimigo, tempoAnimacaoMorte)`
- Colisão com player → `SimpleMove.TakeDamage(dano)`

### 8. Knockback & HitStop
- **KnockBack.cs**: Corrotina aplica força contínua oposta à fonte de dano
- **HitStop.cs**: `Stop()` → `Time.timeScale = 0` temporário (hit pause / game feel)

### 9. Menu & Configurações (`Menu.cs` + `MainMenu.cs`)
- **Pause**: `Escape` → `Time.timeScale = 0/1`, painel `pausePanel`
- **Opções** (`optionsPanel`):
  - Volume: `AudioMixer.SetFloat("volume", value)`
  - Qualidade: `QualitySettings.SetQualityLevel(index)`
  - Fullscreen: `Screen.fullScreen = bool`
  - Resolução: `TMP_Dropdown` populado com `Screen.resolutions` → `Screen.SetResolution()`
- **Morte**: Flag `morte` impede pause durante tela de game over
- **Quit**: `Application.Quit()` (build) / `UnityEditor.EditorApplication.isPlaying = false` (editor)

### 10. Transição de Cenas (`MudarCena.cs`)
- Carrega cenas por nome/índice

### 11. Outras Mecânicas
- **Spawner.cs**: Spawn de inimigos em pontos definidos
- **PlataformMovement.cs**: Plataformas que se movem entre waypoints
- **InteragirComObjetos.cs**: Interação genérica (portas, alavancas, NPCs)
- **Altar.cs**: Mecânica especial (checkpoint, upgrade, etc.)
- **Save.cs**: Persistência de progresso
- **Flecha.cs**: Projétil do arqueiro (velocidade, dano, lifetime)

---

## Assets & Arte

| Pasta | Conteúdo |
|-------|----------|
| `Sprite Sheet/` | Animations do player (idle, walk, jump, attack, hurt, die) e inimigos |
| `PixelArtSteampunk/` | Tilesets, props, UI no estilo pixel art steampunk |
| `TILEMAP EL BOBO castelo 32x32.png` | Tilemap principal do castelo (32×32) + 28 assets derivados (_0 a _27) |
| `barra_hp.png` | Sprite da barra de vida |
| `Soundtrack/` | Músicas de fundo + SFX (ataque, dano, pulo, morte) |
| `Dialogue/` | Sistema de caixas de diálogo / narrativa |

---

## Como Executar

### Pré-requisitos
- **Unity Editor** 2021.3 LTS ou superior (recomendado 2022.3+)
- **.NET Standard 2.1** / .NET 4.x compatible
- **TextMeshPro** (para dropdown de resolução) — incluído via UPM

### Passos
1. Clone o repositório:
   ```bash
   git clone https://github.com/DaniellePereiraaa/ElBobo.git
   ```
2. Abra no **Unity Hub** → "Add project" → selecione a pasta `ElBobo`
3. Aguarde importação de pacotes (Packages/) e compilação de scripts
4. Abra a cena inicial em `Assets/Scenes/`
5. Pressione **Play** no Editor

### Build
- File → Build Settings → Add Open Scenes
- Target Platform: **Windows / Mac / Linux** (Standalone)
- Build and Run

---

## Controles Padrão

| Ação | Tecla |
|------|-------|
| Mover esquerda/direita | ← / →  ou  A / D |
| Pular | Espaço |
| Atacar | Clique esquerdo (Mouse 0) |
| Interagir | F |
| Pausar / Menu | Esc |

---

## Arquitetura & Padrões

| Padrão | Onde Aplicado |
|--------|---------------|
| **Singleton** | `SimpleMove.instance` (acesso global ao player) |
| **Component Pattern** | Cada comportamento em `MonoBehaviour` separado |
| **State Machine (implícito)** | Animator Controllers + triggers (`AtaqueEspada`, `Morrendo`, `Pulando`) |
| **Observer (Unity Events)** | `OnTriggerEnter2D`, `OnCollisionEnter2D` para colisões |
| **Corrotinas** | Knockback, invencibilidade, freeze, animações assíncronas |
| **ScriptableObject (planejado)** | Armas, stats de inimigos, itens |

---

## Roadmap / Melhorias Futuras

- [ ] Sistema de inventário / troca de armas (cetro, arco)
- [ ] Mais tipos de inimigos (mago, tanque, voador)
- [ ] Save/Load completo (PlayerPrefs ou JSON)
- [ ] Pooling de objetos (projéteis, inimigos, particulas)
- [ ] Input System package (novo Unity Input System)
- [ ] Cinemachine para câmera suave
- [ ] Localização (PT/EN/ES)
- [ ] Testes automatizados (Unity Test Framework)

---

## Licença

Projeto educacional / portfolio — livre para estudo e referência.

---

## Autor

Desenvolvido como projeto de **desenvolvimento de jogos em Unity**.

Demonstra:
- Programação C# em Unity (MonoBehaviours, corrotinas, física 2D)
- Arquitetura de sistemas desacoplados (movimento, combate, IA, UI, menu)
- IA de inimigos com raycast (patrol, perseguição, detecção de borda/parede)
- Game feel: knockback, hit stop, invencibilidade, câmera com limites
- Menu completo com configurações de vídeo/áudio
- Pixel art pipeline (tilesets, spritesheets, tilemaps)
- Organização de projeto Unity profissional (Assets/, Prefabs/, Scenes/, Scripts/)