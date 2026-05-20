import numpy as np

# ------------------------------------------------------------
# 1. CEROS Y FASES DE RIEMANN (32 primeros)
# ------------------------------------------------------------
g_32 = np.array([
    14.134725, 21.022040, 25.010858, 30.424876, 32.935062, 37.586178,
    40.918719, 43.327073, 48.005151, 49.773832, 52.970321, 56.446248,
    59.347044, 60.831779, 65.112544, 67.079811, 69.546402, 72.067158,
    75.704691, 77.144840, 79.337375, 82.910381, 84.735493, 87.425275,
    88.809111, 92.491899, 94.651344, 95.870634, 98.831194, 101.317851,
    104.753251, 107.168688
])
fases = np.arctan2(g_32, 0.5)                     # fases de Berry originales
# Rotaciones dependientes de γ: entre 1 y 7 bits
rotaciones = np.array([(int(g * 1e6) % 7) + 1 for g in g_32], dtype=np.uint8)

# ------------------------------------------------------------
# 2. FUNCIÓN DE INTERFERENCIA ESPECTRAL
# ------------------------------------------------------------
def F_N(x_vals, gammas, fases):
    x = np.where(x_vals == 0, 1e-12, x_vals)
    res = np.zeros_like(x, dtype=np.float64)
    for g, a in zip(gammas, fases):
        rho = np.sqrt(0.25 + g**2)
        res += (2.0 * np.sqrt(x) / rho) * np.cos(g * np.log(x) - a)
    return -res

# ------------------------------------------------------------
# 3. NEXUS HASH v2 (NH‑256)
# ------------------------------------------------------------
def nexus_hash_v2(message: str) -> str:
    # --- Estado inicial de 32 bytes desde γ ---
    state = np.array([int(abs(g * 1000) % 256) for g in g_32], dtype=np.uint8)

    # --- Padding Merkle‑Damgård ---
    msg_bytes = message.encode('utf-8')
    bit_length = len(msg_bytes) * 8
    padded = bytearray(msg_bytes)
    padded.append(0x80)                         # bit '1'
    while (len(padded) % 64) != 56:             # relleno de ceros
        padded.append(0x00)
    padded += bit_length.to_bytes(8, 'big')     # longitud en 8 bytes big‑endian

    # --- Procesar bloques de 64 bytes ---
    for i in range(0, len(padded), 64):
        bloque = np.frombuffer(padded[i:i+64], dtype=np.uint8)

        # 8 rondas por bloque
        for _ in range(8):
            # Absorción: suma modular byte a byte
            state = (state + bloque) % 256

            # Evaluación espectral y máscara
            vals = state.astype(np.float64)
            interferencia = F_N(vals, g_32, fases)
            mask = (np.abs(interferencia) * 1e15 % 256).astype(np.uint8)

            # Mezcla XOR
            state ^= mask

            # Rotaciones dependientes de γ
            for j in range(32):
                b = state[j]
                r = rotaciones[j]
                state[j] = ((b >> r) | (b << (8 - r))) & 0xFF

            # Difusión: sumas modulares entre vecinos
            for j in range(1, 32):
                state[j] = (state[j] + state[j-1]) % 256
            state[0] = (state[0] + state[31]) % 256

    return ''.join(format(b, '02x') for b in state)

# ------------------------------------------------------------
# 4. PRUEBA
# ------------------------------------------------------------
msg1 = "Fénix, el universo es coherente."
msg2 = "Fenix, el universo es coherente."      # un tilde de diferencia
print(f"Nexus Hash v2 (1): {nexus_hash_v2(msg1)}")
print(f"Nexus Hash v2 (2): {nexus_hash_v2(msg2)}")
