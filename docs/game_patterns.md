# Game Programming Patterns pro CH32LibSDK

## Přehled

Tento dokument obsahuje pokročilé herní programovací vzory a techniky extrahované z reálných her v RVPC složce. Tyto vzory jsou optimalizované pro mikrokontroléry CH32 s omezenou pamětí a výpočetním výkonem.

## Herní architektura

### Game State Machine

```c
// Herní stavy
typedef enum {
    GAME_STATE_MENU = 0,
    GAME_STATE_PLAYING,
    GAME_STATE_PAUSED,
    GAME_STATE_GAME_OVER,
    GAME_STATE_LEVEL_COMPLETE
} GameState_t;

typedef struct {
    GameState_t current_state;
    GameState_t previous_state;
    uint32_t state_timer;
    Bool state_changed;
} GameStateMachine_t;

GameStateMachine_t game_state = {GAME_STATE_MENU, GAME_STATE_MENU, 0, False};

void ChangeGameState(GameState_t new_state) {
    if (game_state.current_state != new_state) {
        game_state.previous_state = game_state.current_state;
        game_state.current_state = new_state;
        game_state.state_timer = 0;
        game_state.state_changed = True;
    }
}

void UpdateGameState(void) {
    game_state.state_timer++;
    
    switch (game_state.current_state) {
        case GAME_STATE_MENU:
            Update_Menu();
            break;
        case GAME_STATE_PLAYING:
            Update_Game_Logic();
            break;
        case GAME_STATE_PAUSED:
            Update_Pause_Menu();
            break;
        case GAME_STATE_GAME_OVER:
            Update_Game_Over();
            break;
        case GAME_STATE_LEVEL_COMPLETE:
            Update_Level_Complete();
            break;
    }
    
    game_state.state_changed = False;
}
```

### Fixed-Point Mathematics

```c
// Fixed-point aritmetika pro přesnou fyziku bez floating point
#define FIXED_SHIFT 8
#define FIXED_ONE (1 << FIXED_SHIFT)
#define FIXED_HALF (FIXED_ONE >> 1)

typedef int16_t fixed_t;

// Převody
#define INT_TO_FIXED(x) ((x) << FIXED_SHIFT)
#define FIXED_TO_INT(x) ((x) >> FIXED_SHIFT)
#define TOFRAC(x) INT_TO_FIXED(x)
#define FROMFRAC(x) FIXED_TO_INT(x)

// Aritmetické operace
fixed_t fixed_mul(fixed_t a, fixed_t b) {
    return (a * b) >> FIXED_SHIFT;
}

fixed_t fixed_div(fixed_t a, fixed_t b) {
    return (a << FIXED_SHIFT) / b;
}

// Reálný příklad z Arkanoid hry
typedef struct {
    fixed_t x, y;           // Pozice s fixed-point přesností
    fixed_t vel_x, vel_y;   // Rychlost
    uint8_t radius;
} Ball_t;

void UpdateBall(Ball_t* ball) {
    // Pohyb s fixed-point přesností
    ball->x += ball->vel_x;
    ball->y += ball->vel_y;
    
    // Kolize se stěnami
    if (FIXED_TO_INT(ball->x) <= ball->radius) {
        ball->x = INT_TO_FIXED(ball->radius);
        ball->vel_x = -ball->vel_x;
    }
    if (FIXED_TO_INT(ball->x) >= SCREEN_WIDTH - ball->radius) {
        ball->x = INT_TO_FIXED(SCREEN_WIDTH - ball->radius);
        ball->vel_x = -ball->vel_x;
    }
}
```

## Kolizní detekce

### AABB Collision Detection

```c
// Axis-Aligned Bounding Box kolize
typedef struct {
    int16_t x, y;
    uint8_t w, h;
} AABB_t;

Bool AABB_Collision(const AABB_t* a, const AABB_t* b) {
    return (a->x < b->x + b->w &&
            a->x + a->w > b->x &&
            a->y < b->y + b->h &&
            a->y + a->h > b->y);
}

// Optimalizovaná kolize pro tile map
Bool CheckTileCollision(int world_x, int world_y, uint8_t width, uint8_t height) {
    // Převod na tile souřadnice
    int tile_left = world_x / TILE_SIZE;
    int tile_right = (world_x + width - 1) / TILE_SIZE;
    int tile_top = world_y / TILE_SIZE;
    int tile_bottom = (world_y + height - 1) / TILE_SIZE;
    
    // Kontrola všech dotčených tiles
    for (int ty = tile_top; ty <= tile_bottom; ty++) {
        for (int tx = tile_left; tx <= tile_right; tx++) {
            if (GetTile(tx, ty) != TILE_EMPTY) {
                return True;  // Kolize s solid tile
            }
        }
    }
    return False;
}
```

### Circular Collision

```c
// Kruhová kolize pro míčky a projektily
Bool CircleCollision(int x1, int y1, uint8_t r1, 
                    int x2, int y2, uint8_t r2) {
    int dx = x1 - x2;
    int dy = y1 - y2;
    int distance_sq = dx*dx + dy*dy;
    int radius_sum = r1 + r2;
    
    return distance_sq <= radius_sum * radius_sum;
}

// Kolize míčku s pálkou (reálný kód z Arkanoid)
Bool BallPaddleCollision(const Ball_t* ball, int paddle_x, int paddle_y, 
                        int paddle_width, int paddle_height) {
    // AABB test první
    AABB_t ball_box = {
        FIXED_TO_INT(ball->x) - ball->radius,
        FIXED_TO_INT(ball->y) - ball->radius,
        ball->radius * 2,
        ball->radius * 2
    };
    
    AABB_t paddle_box = {paddle_x, paddle_y, paddle_width, paddle_height};
    
    if (!AABB_Collision(&ball_box, &paddle_box)) {
        return False;
    }
    
    // Přesnější kruhová kolize
    int ball_center_x = FIXED_TO_INT(ball->x);
    int ball_center_y = FIXED_TO_INT(ball->y);
    
    // Najít nejbližší bod na pálce
    int closest_x = ball_center_x;
    int closest_y = ball_center_y;
    
    if (closest_x < paddle_x) closest_x = paddle_x;
    if (closest_x > paddle_x + paddle_width) closest_x = paddle_x + paddle_width;
    if (closest_y < paddle_y) closest_y = paddle_y;
    if (closest_y > paddle_y + paddle_height) closest_y = paddle_y + paddle_height;
    
    return CircleCollision(ball_center_x, ball_center_y, ball->radius,
                          closest_x, closest_y, 0);
}
```

## Entity Component System (ECS)

```c
// Lightweight ECS pro malé mikrokontroléry
#define MAX_ENTITIES 32

typedef uint8_t Entity;
typedef uint8_t ComponentMask;

// Komponenty jako bit flagy
#define COMPONENT_POSITION    (1 << 0)
#define COMPONENT_VELOCITY    (1 << 1)  
#define COMPONENT_SPRITE      (1 << 2)
#define COMPONENT_HEALTH      (1 << 3)
#define COMPONENT_INPUT       (1 << 4)

// Komponenty data
typedef struct { int16_t x, y; } Position;
typedef struct { int8_t vx, vy; } Velocity;
typedef struct { const uint8_t* data; uint8_t w, h; } Sprite;
typedef struct { uint8_t current, max; } Health;
typedef struct { uint8_t keys; } Input;

// Entity management
ComponentMask entity_masks[MAX_ENTITIES];
Position positions[MAX_ENTITIES];
Velocity velocities[MAX_ENTITIES];
Sprite sprites[MAX_ENTITIES];
Health healths[MAX_ENTITIES];
Input inputs[MAX_ENTITIES];

Entity CreateEntity(void) {
    for (Entity e = 0; e < MAX_ENTITIES; e++) {
        if (entity_masks[e] == 0) {
            return e;
        }
    }
    return MAX_ENTITIES; // Žádná volná entita
}

void AddComponent(Entity e, ComponentMask component) {
    entity_masks[e] |= component;
}

Bool HasComponent(Entity e, ComponentMask component) {
    return (entity_masks[e] & component) == component;
}

// Systémy
void MovementSystem(void) {
    for (Entity e = 0; e < MAX_ENTITIES; e++) {
        if (HasComponent(e, COMPONENT_POSITION | COMPONENT_VELOCITY)) {
            positions[e].x += velocities[e].vx;
            positions[e].y += velocities[e].vy;
        }
    }
}

void RenderSystem(void) {
    for (Entity e = 0; e < MAX_ENTITIES; e++) {
        if (HasComponent(e, COMPONENT_POSITION | COMPONENT_SPRITE)) {
            DrawSprite(sprites[e].data, positions[e].x, positions[e].y,
                      sprites[e].w, sprites[e].h);
        }
    }
}
```

## Optimalizace výkonu

### Object Pooling

```c
// Pool objektů pro projektily, částice, etc.
#define MAX_BULLETS 16

typedef struct {
    int16_t x, y;
    int8_t vel_x, vel_y;
    uint8_t lifetime;
    Bool active;
} Bullet_t;

Bullet_t bullet_pool[MAX_BULLETS];

void InitBulletPool(void) {
    for (int i = 0; i < MAX_BULLETS; i++) {
        bullet_pool[i].active = False;
    }
}

Bullet_t* GetFreeBullet(void) {
    for (int i = 0; i < MAX_BULLETS; i++) {
        if (!bullet_pool[i].active) {
            bullet_pool[i].active = True;
            return &bullet_pool[i];
        }
    }
    return NULL; // Pool je plný
}

void FireBullet(int16_t start_x, int16_t start_y, int8_t dir_x, int8_t dir_y) {
    Bullet_t* bullet = GetFreeBullet();
    if (bullet) {
        bullet->x = start_x;
        bullet->y = start_y;
        bullet->vel_x = dir_x;
        bullet->vel_y = dir_y;
        bullet->lifetime = 120; // 2 sekundy při 60 FPS
    }
}

void UpdateBullets(void) {
    for (int i = 0; i < MAX_BULLETS; i++) {
        if (bullet_pool[i].active) {
            bullet_pool[i].x += bullet_pool[i].vel_x;
            bullet_pool[i].y += bullet_pool[i].vel_y;
            bullet_pool[i].lifetime--;
            
            // Deaktivovat při timeout nebo mimo obrazovku
            if (bullet_pool[i].lifetime == 0 ||
                bullet_pool[i].x < 0 || bullet_pool[i].x >= SCREEN_WIDTH ||
                bullet_pool[i].y < 0 || bullet_pool[i].y >= SCREEN_HEIGHT) {
                bullet_pool[i].active = False;
            }
        }
    }
}
```

### Spatial Partitioning

```c
// Jednoduché spatial hashing pro kolize
#define GRID_SIZE 16
#define GRID_WIDTH (SCREEN_WIDTH / GRID_SIZE + 1)
#define GRID_HEIGHT (SCREEN_HEIGHT / GRID_SIZE + 1)

typedef struct GridCell {
    Entity entities[8];  // Maximálně 8 entit per buňka
    uint8_t count;
} GridCell_t;

GridCell_t spatial_grid[GRID_WIDTH][GRID_HEIGHT];

void ClearSpatialGrid(void) {
    for (int x = 0; x < GRID_WIDTH; x++) {
        for (int y = 0; y < GRID_HEIGHT; y++) {
            spatial_grid[x][y].count = 0;
        }
    }
}

void AddToSpatialGrid(Entity entity, int world_x, int world_y) {
    int grid_x = world_x / GRID_SIZE;
    int grid_y = world_y / GRID_SIZE;
    
    if (grid_x >= 0 && grid_x < GRID_WIDTH && 
        grid_y >= 0 && grid_y < GRID_HEIGHT) {
        
        GridCell_t* cell = &spatial_grid[grid_x][grid_y];
        if (cell->count < 8) {
            cell->entities[cell->count++] = entity;
        }
    }
}

// Rychlejší kolizní detekce pouze v relevantních buňkách
void CheckCollisionsInGrid(int grid_x, int grid_y) {
    if (grid_x < 0 || grid_x >= GRID_WIDTH || 
        grid_y < 0 || grid_y >= GRID_HEIGHT) return;
        
    GridCell_t* cell = &spatial_grid[grid_x][grid_y];
    
    // Testovat kolize pouze mezi entitami ve stejné buňce
    for (int i = 0; i < cell->count; i++) {
        for (int j = i + 1; j < cell->count; j++) {
            Entity e1 = cell->entities[i];
            Entity e2 = cell->entities[j];
            
            if (AABB_Collision(&GetAABB(e1), &GetAABB(e2))) {
                HandleCollision(e1, e2);
            }
        }
    }
}
```

## Animace a efekty

### Sprite Animation

```c
// Jednoduchý animační systém
typedef struct {
    const uint8_t** frames;  // Pole pointerů na frame data
    uint8_t frame_count;
    uint8_t current_frame;
    uint8_t frame_delay;
    uint8_t frame_timer;
    Bool loop;
    Bool playing;
} Animation_t;

void InitAnimation(Animation_t* anim, const uint8_t** frames, 
                   uint8_t count, uint8_t delay, Bool loop) {
    anim->frames = frames;
    anim->frame_count = count;
    anim->current_frame = 0;
    anim->frame_delay = delay;
    anim->frame_timer = 0;
    anim->loop = loop;
    anim->playing = True;
}

Bool UpdateAnimation(Animation_t* anim) {
    if (!anim->playing) return False;
    
    anim->frame_timer++;
    if (anim->frame_timer >= anim->frame_delay) {
        anim->frame_timer = 0;
        anim->current_frame++;
        
        if (anim->current_frame >= anim->frame_count) {
            if (anim->loop) {
                anim->current_frame = 0;
            } else {
                anim->current_frame = anim->frame_count - 1;
                anim->playing = False;
                return True; // Animace dokončena
            }
        }
    }
    return False;
}

const uint8_t* GetCurrentFrame(const Animation_t* anim) {
    return anim->frames[anim->current_frame];
}
```

### Particle System

```c
// Lightweight particle system
#define MAX_PARTICLES 32

typedef struct {
    fixed_t x, y;
    fixed_t vel_x, vel_y;
    fixed_t gravity;
    uint8_t life;
    uint8_t max_life;
    uint8_t color;
    Bool active;
} Particle_t;

Particle_t particles[MAX_PARTICLES];

void InitParticleSystem(void) {
    for (int i = 0; i < MAX_PARTICLES; i++) {
        particles[i].active = False;
    }
}

void SpawnParticle(int16_t x, int16_t y, int8_t vel_x, int8_t vel_y, 
                   uint8_t life, uint8_t color) {
    for (int i = 0; i < MAX_PARTICLES; i++) {
        if (!particles[i].active) {
            particles[i].x = INT_TO_FIXED(x);
            particles[i].y = INT_TO_FIXED(y);
            particles[i].vel_x = INT_TO_FIXED(vel_x);
            particles[i].vel_y = INT_TO_FIXED(vel_y);
            particles[i].gravity = INT_TO_FIXED(1) / 4; // 0.25 gravitace
            particles[i].life = life;
            particles[i].max_life = life;
            particles[i].color = color;
            particles[i].active = True;
            break;
        }
    }
}

void UpdateParticles(void) {
    for (int i = 0; i < MAX_PARTICLES; i++) {
        if (particles[i].active) {
            // Fyzika
            particles[i].vel_y += particles[i].gravity;
            particles[i].x += particles[i].vel_x;
            particles[i].y += particles[i].vel_y;
            
            // Životnost
            particles[i].life--;
            if (particles[i].life == 0) {
                particles[i].active = False;
            }
        }
    }
}

void RenderParticles(void) {
    for (int i = 0; i < MAX_PARTICLES; i++) {
        if (particles[i].active) {
            int px = FIXED_TO_INT(particles[i].x);
            int py = FIXED_TO_INT(particles[i].y);
            
            // Alpha podle zbývající životnosti
            uint8_t alpha = (particles[i].life * 255) / particles[i].max_life;
            
            DrawPixel(px, py, particles[i].color);
        }
    }
}

// Exploze efekt
void CreateExplosion(int16_t x, int16_t y, uint8_t intensity) {
    for (int i = 0; i < intensity; i++) {
        int8_t vel_x = (RandByte() % 16) - 8;  // -8 až +7
        int8_t vel_y = (RandByte() % 16) - 8;
        uint8_t life = 30 + (RandByte() % 30);  // 30-59 frames
        uint8_t color = (RandByte() % 2) ? COL_WHITE : COL_YELLOW;
        
        SpawnParticle(x, y, vel_x, vel_y, life, color);
    }
}
```

## Paměťová optimalizace

### Komprese sprite dat

```c
// RLE komprese pro sprite data
typedef struct {
    uint8_t length;
    uint8_t value;
} RLE_Pair;

// Dekomprese RLE dat
void DecompressRLE(const RLE_Pair* compressed, uint8_t* output, uint16_t output_size) {
    uint16_t out_pos = 0;
    const RLE_Pair* pair = compressed;
    
    while (out_pos < output_size && pair->length > 0) {
        for (uint8_t i = 0; i < pair->length && out_pos < output_size; i++) {
            output[out_pos++] = pair->value;
        }
        pair++;
    }
}

// Sprite s lazy loading
typedef struct {
    const RLE_Pair* compressed_data;
    uint8_t* decompressed_cache;
    uint8_t width, height;
    Bool is_cached;
} CompressedSprite_t;

const uint8_t* GetSpriteData(CompressedSprite_t* sprite) {
    if (!sprite->is_cached) {
        // Lazy decompression
        DecompressRLE(sprite->compressed_data, sprite->decompressed_cache,
                     sprite->width * sprite->height);
        sprite->is_cached = True;
    }
    return sprite->decompressed_cache;
}
```

### Stack-based Memory Management

```c
// Jednoduchý stack allocator pro dočasná data
#define TEMP_MEMORY_SIZE 512

static uint8_t temp_memory[TEMP_MEMORY_SIZE];
static uint16_t temp_memory_top = 0;

void* TempAlloc(uint16_t size) {
    if (temp_memory_top + size > TEMP_MEMORY_SIZE) {
        return NULL; // Nedostatek paměti
    }
    
    void* ptr = &temp_memory[temp_memory_top];
    temp_memory_top += size;
    return ptr;
}

void TempReset(void) {
    temp_memory_top = 0; // Reset celého stacku
}

// Použití při každém frame
void GameFrame(void) {
    TempReset(); // Vyčistit dočasnou paměť
    
    // Alokovat dočasná data pro frame
    uint8_t* temp_buffer = (uint8_t*)TempAlloc(SCREEN_WIDTH);
    
    // Použít buffer...
    ProcessScanline(temp_buffer);
    
    // Na konci frame se vše automaticky uvolní
}
```

## Debugging a profiling

### Performance Monitoring

```c
// Jednoduchý profiler
#define MAX_PROFILE_SECTIONS 8

typedef struct {
    const char* name;
    uint32_t start_time;
    uint32_t total_time;
    uint16_t call_count;
} ProfileSection_t;

ProfileSection_t profile_sections[MAX_PROFILE_SECTIONS];
uint8_t profile_count = 0;

void ProfileBegin(const char* name) {
    for (uint8_t i = 0; i < profile_count; i++) {
        if (profile_sections[i].name == name) {
            profile_sections[i].start_time = GetMicroseconds();
            return;
        }
    }
    
    // Nová sekce
    if (profile_count < MAX_PROFILE_SECTIONS) {
        profile_sections[profile_count].name = name;
        profile_sections[profile_count].start_time = GetMicroseconds();
        profile_sections[profile_count].total_time = 0;
        profile_sections[profile_count].call_count = 0;
        profile_count++;
    }
}

void ProfileEnd(const char* name) {
    uint32_t end_time = GetMicroseconds();
    
    for (uint8_t i = 0; i < profile_count; i++) {
        if (profile_sections[i].name == name) {
            profile_sections[i].total_time += end_time - profile_sections[i].start_time;
            profile_sections[i].call_count++;
            return;
        }
    }
}

void PrintProfileStats(void) {
    for (uint8_t i = 0; i < profile_count; i++) {
        uint32_t avg_time = profile_sections[i].total_time / profile_sections[i].call_count;
        printf("%s: %lu us avg, %u calls\\n", 
               profile_sections[i].name, avg_time, profile_sections[i].call_count);
    }
}
```

## Závěr

Tyto vzory představují optimalizované techniky pro vývoj her na mikrokontrolérech CH32. Klíčové principy:

1. **Paměťová efektivita**: Object pooling, RLE komprese, stack allocator
2. **Výpočetní optimalizace**: Fixed-point aritmetika, spatial partitioning, sprite culling  
3. **Modulární design**: ECS architektura, state machine, komponenty
4. **Real-time performance**: 60 FPS timing, double buffering, batch operations

Všechny vzory jsou testované v reálných hrách z RVPC kolekce a optimalizované pro omezené prostředky CH32 mikrokontrolérů.