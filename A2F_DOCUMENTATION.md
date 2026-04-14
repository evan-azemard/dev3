# 📚 Documentation Complète de l'A2F (Authentification à Deux Facteurs)

> [!TIP]

## 📖 Table des matières
1. [Concepts fondamentaux](#concepts-fondamentaux)
2. [Architecture générale](#architecture-générale)
3. [Fichiers Backend](#fichiers-backend)
4. [Fichiers Frontend](#fichiers-frontend)
5. [Flux complet étape par étape](#flux-complet-étape-par-étape)
6. [Database](#database)

---

## Concepts fondamentaux

### 🔐 OTP vs A2F : C'est quoi la différence?

#### **OTP (One-Time Password - Code à usage unique)**
- 💡 C'est le **code numérique** que vous tapez (ex: 123456)
- ⏱️ Valide pendant 30 secondes seulement
- 🔄 Généré dynamiquement par une app comme Google Authenticator
- 📱 Synchronisé via l'algorithme TOTP (Time-based OTP)

**Exemple d'OTP:**
```
Temps t=0s   → OTP = 453621
Temps t=10s  → OTP = 453621 (même)
Temps t=30s  → OTP = 782014 (changé!)
```

#### **A2F (Authentification à Deux Facteurs)**
- 🎯 C'est la **fonctionnalité complète** qui protège votre compte
- 🔗 Combine 2 méthodes d'authentification:
  1. **Quelque chose que vous connaissez** = mot de passe
  2. **Quelque chose que vous avez** = téléphone (app TOTP)
- 🛡️ Même si quelqu'un connaît votre mdp, il ne peut pas se connecter sans votre téléphone

**Flux A2F:**
```
Utilisateur            Backend              Téléphone
     |
     |--[email + mdp]--> ✅ Valide
     |
     |                   ❌ "A2F activée! Besoin d'OTP"
     |<--[Token temp]--
     |
     | ⏱️ Utilisateur regarde son téléphone
     | 📱 Google Authenticator affiche: 453621
     |
     |--[OTP 453621]---> ✅ Valide (TOTP correct)
     |
     |<--[Token final]-- ✅ Connecté!
```

---

## Architecture générale

### 🏗️ Vue d'ensemble du système

```
┌─────────────────────────────────────────────────────────────────┐
│                        FRONTEND (React)                          │
├─────────────────────────────────────────────────────────────────┤
│  A2FSection.tsx    → Interface utilisateur                       │
│  a2f.usecase.ts    → Appels API Redux Thunk                     │
│  a2f.slice.ts      → État Redux (QR, secret, codes de backup)   │
└──────────────────────┬──────────────────────────────────────────┘
                       │
                       │ HTTP (credentials: include)
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│                        BACKEND (Express)                         │
├─────────────────────────────────────────────────────────────────┤
│  a2f.controller.ts                                              │
│  ├─ qrCode()        → GET /api/a2f/qrcode                      │
│  ├─ enableA2F()     → POST /api/a2f/enable                     │
│  ├─ verifyOtp()     → POST /api/a2f/verify-otp                 │
│  └─ disableA2F()    → POST /api/a2f/disable                    │
│                                                                  │
│  EnableA2FUseCase      → Logique d'activation A2F              │
│  VerifyOtpUseCase      → Logique de vérification OTP           │
│  LoginUseCase          → Logique de login (avec A2F)           │
│                                                                  │
│  QrCodeGenerator       → Génère les QR codes                   │
│  PrismaUserRepository  → Accès à la DB (User table)            │
│  OtpBackupCodeRepository → Codes de backup (codes de secours)  │
└──────────────────────┬──────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│                    DATABASE (PostgreSQL)                         │
├─────────────────────────────────────────────────────────────────┤
│  User table:                                                    │
│  - otpSecret (String)  → Secret partagé (qrcode + authentateur) │
│  - otpEnabled (Int)    → 0 ou 1 (disabled/enabled)             │
│                                                                  │
│  OtpBackupCode table:                                          │
│  - codes (JSON)        → Codes de secours hashés (bcrypt)      │
│  - nb_code_used (Int)  → Nombre de codes utilisés              │
└─────────────────────────────────────────────────────────────────┘
```

### 📚 Dépendances principales

**Backend:**
- `otplib` - Génère secrets TOTP, vérifie OTP
- `qrcode` - Convertit URI en image QR
- `bcrypt` - Hash des codes de backup
- `@prisma/client` - ORM pour la base de données
- `express` - Framework web
- `jsonwebtoken` - Gestion des JWT

**Frontend:**
- `@reduxjs/toolkit` - État global
- `react` - Composants UI
- `@mui/material` - Composants Material Design

---

## Fichiers Backend

### 1️⃣ `a2f.controller.ts` - Les endpoints HTTP

**Chemin:** `EventHub-Backend/src/api/controllers/a2f.controller.ts`

**Endpoints:**
- `GET /api/a2f/qrcode` → Génère QR code
- `POST /api/a2f/enable` → Active A2F avec vérification OTP
- `POST /api/a2f/verify-otp` → Vérifie OTP lors du login
- `POST /api/a2f/disable` → Désactive A2F

#### Code complet: `qrCode()`
```typescript
export const qrCode = async (
    req: Request,
    res: Response,
    next: NextFunction
): Promise<any> => {
    try {
        console.log('[A2F Controller] GET /a2f/qrcode - Début du traitement');
        
        // 1. Vérifier que l'utilisateur est authentifié
        if (!req.user) {
            console.error('Utilisateur non authentifié');
            return res.jsonError("Vous n'êtes pas connecté", 401);
        }

        // 2. Afficher les clés disponibles dans otplib
        console.log('otplib keys:', Object.keys(otplib || {}));
        
        // 3. Générer un secret aléatoire
        // Ce secret sera utilisé par l'authenticateur du téléphone
        const secret = otplib.generateSecret();
        console.log('Secret généré:', secret?.substring(0, 10) + '...');
        
        // 4. Utiliser le QRCode Generator pour créer l'image
        const qrCodeGenerator = container.resolve<any>('qrCodeGenerator');
        const qrCodeData = await qrCodeGenerator.generate(req.user.email, secret);
        
        // 5. Retourner l'image QR + secret (le secret doit être stocké client temporairement)
        return res.jsonSuccess({ qrCode: qrCodeData }, 200);
    } catch (error: any) {
        console.error('Erreur dans qrCode controller:', error?.message);
        next(error);
    }
};
```

#### Code complet: `enableA2F()`
```typescript
export const enableA2F = async (
    req: Request,
    res: Response,
    next: NextFunction
): Promise<any> => {
    try {
        console.log('[A2F Controller] POST /a2f/enable - Début du traitement');
        
        // 1. Vérifier authentification
        if (!req.user) {
            return res.jsonError("Vous n'êtes pas connecté", 401);
        }

        // 2. Récupérer l'OTP token et le secret depuis le body
        const { otpToken, secret } = req.body;
        
        if (!otpToken || !secret) {
            return res.jsonError("otpToken et secret sont requis", 400);
        }

        // 3. Exécuter le UseCase qui valide le OTP et active A2F
        const useCase = container.resolve<any>('EnableA2FUseCase');
        const result = await useCase.execute({
            userId: req.user.userId,
            otpToken,
            secret
        });
        
        // 4. Retourner les codes de backup (à afficher une seule fois!)
        return res.jsonSuccess({ backupCodes: result.backupCodes }, 200);
    } catch (error: any) {
        console.error('Erreur:', error?.message);
        
        if (error?.message === 'Code OTP invalide') {
            return res.jsonError(error.message, 400);
        }
        next(error);
    }
};
```

### 2️⃣ `EnableA2FUseCase.ts` - Logique d'activation

**Chemin:** `EventHub-Backend/src/application/usecases/EnableA2FUseCase.ts`

**Responsabilités:**
1. Chercher l'utilisateur par ID
2. Vérifier que le OTP est valide
3. Mettre à jour `otpEnabled = 1` en DB
4. Générer 6 codes de backup (hashés en bcrypt)
5. Sauvegarder les codes de backup en DB

```typescript
interface EnableA2FInput {
    userId: string;
    otpToken: string;        // Code OTP saisi par l'utilisateur
    secret: string;           // Secret unique pour cet user
}

interface EnableA2FResult {
    backupCodes: string[];   // Les 6 codes de backup EN CLAIR (à afficher une fois)
}

@injectable()
export class EnableA2FUseCase {
    constructor(
        @inject('UserRepository')
        private userRepository: UserRepositoryInterface,
        @inject('OtpBackupCodeRepository')
        private otpBackupCodeRepository: IOtpBackupCodeRepository
    ) {}

    async execute(input: EnableA2FInput): Promise<EnableA2FResult> {
        // 1. Récupérer l'utilisateur
        const user = await this.userRepository.findById(input.userId);
        if (!user) {
            throw new Error('Utilisateur introuvable');
        }

        // 2. ⭐ VÉRIFIER LE OTPTOKEN
        // otplib.verify() prend le token et le secret à vérifier
        // Il va vérifier que le code est correct et pas expiré
        const result = await otplib.verify({
            token: input.otpToken,      // ex: "453621"
            secret: input.secret        // secret unique
        });

        if (!result.valid) {
            throw new Error('Code OTP invalide');
        }

        // 3. ✅ OTP correct! Activer A2F en DB
        // Mettre à jour: otpSecret (pour les futures vérifications) et otpEnabled = 1
        await this.userRepository.updateOtp(input.userId, input.secret, 1);

        // 4. Générer 6 codes de backup (codes de secours)
        // Si l'utilisateur perd son téléphone, ces codes lui permettent de se connecter
        const rawCodes = generateBackupCodes(6);  // ex: ["ABC12345", "XYZ98765", ...]
        
        // 5. HASHER les codes avec bcrypt
        // On ne stocke JAMAIS les codes en clair en base de données!
        const hashedCodes = await Promise.all(
            rawCodes.map(c => bcrypt.hash(c, 10))
        );

        // 6. Sauvegarder les codes hashés en DB
        const entity = new OtpBackupCode({
            user_id: input.userId,
            codes: JSON.stringify(hashedCodes),
            nb_code_used: 0,
            nb_consecutive_tests: 0
        });

        const existing = await this.otpBackupCodeRepository.findByUserId(input.userId);
        if (existing) {
            entity.props.id = existing.props.id;
            await this.otpBackupCodeRepository.update(entity);
        } else {
            await this.otpBackupCodeRepository.save(entity);
        }

        // 7. ✅ Retourner les codes EN CLAIR (pour que l'utilisateur les sauvegarde)
        return { backupCodes: rawCodes };
    }
}
```

### 3️⃣ `VerifyOtpUseCase.ts` - Vérifier OTP lors du login

**Chemin:** `EventHub-Backend/src/application/usecases/VerifyOtpUseCase.ts`

**Responsabilités:**
1. Vérifier que le tempToken est valide
2. Vérifier que c'est un token "pendingOtp"
3. Vérifier que le OTP est correct
4. Générer un token final pour se connecter

```typescript
interface VerifyOtpInput {
    otpToken: string;        // Code OTP saisi par l'utilisateur
    tempToken: string;       // JWT temporaire (valide 5 min)
}

@injectable()
export class VerifyOtpUseCase {
    constructor(
        @inject('UserRepository')
        private userRepository: UserRepositoryInterface
    ) {}

    async execute(input: VerifyOtpInput): Promise<VerifyOtpResult> {
        // 1. Décoder le tempToken
        const secret = process.env.JWT_SECRET || 'secret';
        let payload: any;
        try {
            payload = jwt.verify(input.tempToken, secret);
        } catch {
            throw new Error('Token temporaire invalide ou expiré');
        }

        // 2. Vérifier que c'est bien un token "pendingOtp"
        if (!payload.pendingOtp) {
            throw new Error('Token ne correspond pas à un challenge OTP');
        }

        // 3. Récupérer l'utilisateur
        const user = await this.userRepository.findById(payload.userId);
        if (!user) {
            throw new Error('Utilisateur introuvable');
        }

        // 4. ⭐ VÉRIFIER LE OTPTOKEN AVEC LE SECRET STOCKÉ
        const result = await otplib.verify({
            token: input.otpToken,     // ex: "453621"
            secret: user.otpSecret     // secret qu'on a stocké en DB
        });

        if (!result.valid) {
            throw new Error('Code OTP invalide');
        }

        // 5. ✅ OTP correct! Générer un token final pour se connecter
        const token = jwt.sign(
            { userId: user.id, email: user.email, name: user.name },
            secret,
            { expiresIn: '24h' }
        );

        return {
            token,
            userId: user.id!,
            email: user.email,
            name: user.name
        };
    }
}
```

### 4️⃣ `LoginUseCase.ts` - Gestion du login avec A2F

**Chemin:** `EventHub-Backend/src/application/usecases/LoginUseCase.ts`

**Responsabilités:**
1. Vérifier email/mdp
2. Si A2F activée → générer tempToken et demander OTP
3. Si A2F non activée → générer token final

```typescript
async execute(dto: LoginDTO): Promise<LoginResponse> {
    // 1. Vérifier email/mdp
    const user = await this.userRepository.findByEmail(dto.email);
    if (!user) {
        throw new UnauthorizedError('Email ou mot de passe incorrect');
    }

    const isPasswordValid = await bcrypt.compare(dto.password, user.password);
    if (!isPasswordValid) {
        throw new UnauthorizedError('Email ou mot de passe incorrect');
    }

    const jwtSecret = process.env.JWT_SECRET || 'secret';

    // 2. Vérifier si A2F est activée
    if (user.otpEnabled === 1) {
        // ⭐ A2F ACTIVÉE!
        // On ne connecte PAS tout de suite
        // On crée un token TEMPORAIRE valide 5 min pour vérifier l'OTP
        
        const tempToken = jwt.sign(
            { 
                userId: user.id, 
                email: user.email, 
                pendingOtp: true    // ← Flag qui indique qu'on attend un OTP
            },
            jwtSecret,
            { expiresIn: '5m' }
        );
        
        // On renvoie le tempToken pour que le client le stocke
        return {
            email: user.email,
            name: user.name,
            requiresOtp: true,      // ← Signal pour le frontend
            tempToken
        };
    }

    // 3. A2F NON activée → connexion normale
    const token = jwt.sign(
        { userId: user.id, email: user.email, name: user.name },
        jwtSecret,
        { expiresIn: '24h' }
    );

    return {
        token,
        userId: user.id!,
        email: user.email,
        name: user.name
    };
}
```

### 5️⃣ `QrCodeGenerator.ts` - Génère les QR codes

**Chemin:** `EventHub-Backend/src/shared/utils/qr-code-generator.ts`

**Responsabilités:**
1. Prendre un secret + username
2. Créer une URI au format TOTP
3. Convertir l'URI en image QR

```typescript
export class QrCodeGenerator implements iQrCode {
    constructor(private readonly appName: string) {}
    
    async generate(username: string, secret: string) {
        // 1. Utiliser otplib pour créer une URI au format TOTP
        // Cette URI contient le secret + les infos pour l'authenticateur
        const keyUri = otplib.generateURI({
            issuer: this.appName,       // "EventHub"
            label: username,             // email de l'utilisateur
            secret: secret               // secret unique
        });

        // Format généré:
        // otpauth://totp/EventHub:user@example.com?secret=JBSWY3DPEBLW64TMMQ&issuer=EventHub

        // 2. Convertir l'URI en image QR en base64
        const image = await qrcode.toDataURL(keyUri);
        // Retourne: "data:image/png;base64,iVBORw0KGgo..."

        // 3. Retourner l'image + les infos
        return Promise.resolve({ 
            image,           // Image QR en base64
            username,
            secret           // Le secret (le client doit le stocker!)
        });
    }
}
```

### 6️⃣ `a2fRoutes.ts` - Configuration des routes

**Chemin:** `EventHub-Backend/src/api/routes/a2fRoutes.ts`

```typescript
import { Router } from "express";
import { authMiddleware } from "../middlewares/authMiddleware";
import { qrCode, enableA2F, verifyOtp, verifyBackupCode, disableA2F } from "../controllers/a2f.controller";

const router = Router();

// ❌ Routes PUBLIC (pas de session cookie nécessaire)
// Ces routes sont appelées lors du login quand l'utilisateur tape l'OTP
router.post('/verify-otp', verifyOtp);              // Login avec A2F
router.post('/verify-backup-code', verifyBackupCode); // Login avec code de backup

// ✅ Routes PROTÉGÉES (besoin d'être authentifié)
// Ces routes nécessitent une session valide
router.use(authMiddleware);
router.get('/qrcode', qrCode);    // Générer QR code (setup)
router.post('/enable', enableA2F);  // Activer A2F (verification OTP)
router.post('/disable', disableA2F); // Désactiver A2F

export { router as a2fRouter };
```

---

## Fichiers Frontend

### 1️⃣ `a2f.slice.ts` - État Redux

**Chemin:** `EventHubFrontend/src/modules/profile/core/slice/a2f.slice.ts`

```typescript
export type A2FState = {
    isLoading: boolean;          // ⏳ En cours de chargement?
    error: string | null;         // ❌ Message d'erreur
    qrCodeImage: string | null;   // 🔲 Image QR en base64
    secret: string | null;        // 🔑 Secret (à ne pas perdre!)
    backupCodes: string[] | null; // 🔐 Codes de backup
    enabled: boolean;             // ✅ A2F activée?
    success: string | null;       // ✅ Message de succès
};

const initialState: A2FState = {
    isLoading: false,
    error: null,
    qrCodeImage: null,
    secret: null,
    backupCodes: null,
    enabled: false,
    success: null,
};

export const a2fSlice = createSlice({
    name: "a2f",
    initialState,
    reducers: {
        // 📥 Récupération du QR code
        fetchQrStart: (state) => {
            state.isLoading = true;
            state.error = null;
        },
        fetchQrSuccess: (state, action: PayloadAction<{ image: string; secret: string }>) => {
            state.isLoading = false;
            state.qrCodeImage = action.payload.image;
            state.secret = action.payload.secret;  // Stock le secret!
        },
        fetchQrFailure: (state, action: PayloadAction<string>) => {
            state.isLoading = false;
            state.error = action.payload;
        },

        // 🔐 Activation A2F
        enableStart: (state) => {
            state.isLoading = true;
            state.error = null;
        },
        enableSuccess: (state, action: PayloadAction<string[]>) => {
            state.isLoading = false;
            state.enabled = true;
            state.backupCodes = action.payload;  // Stock les codes de backup!
            state.success = "A2F activée avec succès!";
        },
        enableFailure: (state, action: PayloadAction<string>) => {
            state.isLoading = false;
            state.error = action.payload;
        },

        // Les autres reducers...
    },
});
```

### 2️⃣ `a2f.usecase.ts` - Actions Redux Thunk

**Chemin:** `EventHubFrontend/src/modules/profile/core/usecases/a2f.usecase.ts`

#### Action 1: Récupérer le QR code
```typescript
export const fetchQrCodeUsecase = () => async (dispatch: AppDispatch) => {
    dispatch(fetchQrStart());
    try {
        console.log('🔵 [A2F] Tentative de récupération du QR code...');
        
        // 📤 Appel API: GET /api/a2f/qrcode
        const res = await fetch(`${API}/a2f/qrcode`, {
            method: "GET",
            credentials: "include",  // Envoyer le cookie d'authentification
        });
        
        console.log('📊 Status:', res.status);
        
        // Vérifier le statut
        if (!res.ok) {
            const json = await res.json();
            dispatch(fetchQrFailure(json.message));
            return;
        }
        
        // 📥 Parser la réponse
        const json = await res.json();
        const { image, secret } = json.data.qrCode;
        
        console.log('✅ QR Code reçu');
        
        // 💾 Stocker en Redux
        dispatch(fetchQrSuccess({ image, secret }));
        
    } catch (e: unknown) {
        const errorMsg = e instanceof Error ? e.message : "Erreur inconnue";
        console.error('💥 Erreur:', errorMsg);
        dispatch(fetchQrFailure(errorMsg));
    }
};
```

#### Action 2: Activer A2F
```typescript
export const enableA2FUsecase =
    (otpToken: string, secret: string) => async (dispatch: AppDispatch) => {
        dispatch(enableStart());
        try {
            console.log('🔵 [A2F] Activation en cours...');
            
            // 📤 Appel API: POST /api/a2f/enable
            const res = await fetch(`${API}/a2f/enable`, {
                method: "POST",
                credentials: "include",
                headers: { "Content-Type": "application/json" },
                body: JSON.stringify({ 
                    otpToken,   // ex: "453621" (code saisi par l'utilisateur)
                    secret      // ex: "JBSWY3DPEBLW64TMMQ" (qui vient du QR)
                }),
            });
            
            console.log('📊 Status:', res.status);
            
            if (!res.ok) {
                const json = await res.json();
                dispatch(enableFailure(json.message));
                return;
            }
            
            // 📥 Parser la réponse
            const json = await res.json();
            const { backupCodes } = json.data;
            
            console.log('✅ A2F activée!');
            
            // 💾 Stocker les codes de backup en Redux
            // ⚠️ Ces codes ne s'affichent qu'UNE SEULE FOIS!
            dispatch(enableSuccess(backupCodes));
            
        } catch (e: unknown) {
            const errorMsg = e instanceof Error ? e.message : "Erreur inconnue";
            dispatch(enableFailure(errorMsg));
        }
    };
```

### 3️⃣ `A2FSection.tsx` - Interface utilisateur

**Chemin:** `EventHubFrontend/src/modules/profile/components/sections/A2FSection.tsx`

```typescript
/**
 * Composant qui affiche la section A2F sur la page de profil
 */
export const A2FSection = () => {
    const dispatch = useAppDispatch();
    const a2f = useSelector((state: AppState) => state.a2f);
    const [otpToken, setOtpToken] = useState("");
    const [step, setStep] = useState<"idle" | "setup">("idle");

    // 📍 ÉTAPE 1: Utilisateur clique sur "Activer A2F"
    const handleStartSetup = () => {
        console.log('[A2F UI] Clique sur "Activer A2F"');
        setStep("setup");
        
        // 📤 Dispatch l'action pour récupérer le QR code
        dispatch(fetchQrCodeUsecase());
    };

    // 📍 ÉTAPE 2: Utilisateur envoie le code OTP qu'il vient de taper
    const handleConfirmEnable = (e: React.FormEvent) => {
        e.preventDefault();
        console.log('[A2F UI] Soumission du formulaire');
        console.log('OTP saisi:', otpToken);
        console.log('Secret:', a2f.secret);
        
        if (!a2f.secret) {
            console.error('Secret manquant!');
            return;
        }
        
        // 📤 Dispatch l'action pour activer A2F
        dispatch(enableA2FUsecase(otpToken, a2f.secret));
        setOtpToken("");
    };

    return (
        <Paper variant="outlined" sx={{ p: 3, mt: 4 }}>
            {/* En-tête */}
            <Box sx={{ display: "flex", alignItems: "center", gap: 1, mb: 2 }}>
                <ShieldIcon color={a2f.enabled ? "success" : "action"} />
                <Typography variant="h6">Double Authentification (A2F)</Typography>
                {a2f.enabled && (
                    <Chip label="Activée" color="success" size="small" />
                )}
            </Box>

            {/* Affichage des erreurs */}
            {a2f.error && (
                <Alert severity="error" sx={{ mb: 2 }}>
                    {a2f.error}
                </Alert>
            )}

            {/* Affichage des succès */}
            {a2f.success && (
                <Alert severity="success" sx={{ mb: 2 }}>
                    {a2f.success}
                </Alert>
            )}

            {/* AFFICHER LES CODES DE BACKUP (une fois l'A2F activée) */}
            {a2f.backupCodes && a2f.enabled && (
                <Alert severity="warning" sx={{ mb: 3 }}>
                    <Typography variant="subtitle2">
                        ⚠️ Codes de secours - à conserver en lieu sûr!
                    </Typography>
                    <List dense>
                        {a2f.backupCodes.map((code, i) => (
                            <ListItem key={i} disablePadding>
                                <Typography sx={{ fontFamily: "monospace" }}>
                                    {code}
                                </Typography>
                            </ListItem>
                        ))}
                    </List>
                </Alert>
            )}

            {/* ÉTAT: A2F ACTIVÉE */}
            {a2f.enabled && !a2f.backupCodes && (
                <Box>
                    <Typography variant="body2">
                        ✅ La double authentification est activée.
                    </Typography>
                    <Button
                        variant="outlined"
                        color="error"
                        onClick={() => dispatch(disableA2FUsecase())}
                        size="small"
                        sx={{ mt: 1 }}
                    >
                        Désactiver
                    </Button>
                </Box>
            )}

            {/* ÉTAT: CHOIX D'ACTIVER A2F */}
            {!a2f.enabled && step === "idle" && (
                <Box>
                    <Typography variant="body2" color="text.secondary">
                        Activez la double authentification pour sécuriser votre compte.
                    </Typography>
                    <Button
                        variant="contained"
                        onClick={handleStartSetup}
                        disabled={a2f.isLoading}
                        sx={{ mt: 1 }}
                    >
                        Activer A2F
                    </Button>
                </Box>
            )}

            {/* ÉTAT: AFFICHAGE DU QR CODE + FORMULAIRE OTP */}
            {!a2f.enabled && step === "setup" && (
                <Box>
                    {/* Afficher le QR code pendant le chargement ou après */}
                    {a2f.qrCodeImage ? (
                        <>
                            <Typography variant="body2" sx={{ mb: 2 }}>
                                📱 Scannez ce QR code avec Google Authenticator ou Authy
                            </Typography>
                            <Box sx={{ py: 2 }}>
                                <img src={a2f.qrCodeImage} alt="QR Code" style={{ maxWidth: 250 }} />
                            </Box>

                            {/* Formulaire pour saisir le code OTP */}
                            <form onSubmit={handleConfirmEnable}>
                                <TextField
                                    label="Code OTP"
                                    placeholder="123456"
                                    value={otpToken}
                                    onChange={(e) => setOtpToken(e.target.value)}
                                    maxLength="6"
                                    disabled={a2f.isLoading}
                                    fullWidth
                                    margin="normal"
                                />
                                <Button
                                    type="submit"
                                    variant="contained"
                                    disabled={a2f.isLoading || otpToken.length !== 6}
                                    fullWidth
                                    sx={{ mt: 1 }}
                                >
                                    {a2f.isLoading ? "Vérification..." : "Confirmer"}
                                </Button>
                            </form>
                        </>
                    ) : (
                        <Box sx={{ textAlign: "center", py: 2 }}>
                            <CircularProgress />
                            <Typography>Génération du QR code...</Typography>
                        </Box>
                    )}
                </Box>
            )}
        </Paper>
    );
};
```

---

## Flux complet étape par étape

### 🔄 Scénario 1: Activation de l'A2F

```
┌─────────────────────────────────────────────────────────────────────┐
│                     UTILISATEUR ACTIVANT A2F                         │
└─────────────────────────────────────────────────────────────────────┘

ÉTAPE 1: Utilisateur clique sur "Activer A2F"
┌──────────────────────────────────────────────────────────────────┐
│ Frontend (React)                   Backend (Node.js)             │
│                                                                   │
│ [Clique]                                                          │
│   ↓                                                               │
│ handleStartSetup()                                               │
│   ↓                                                               │
│ dispatch(fetchQrCodeUsecase())                                   │
│   ↓                                                               │
│ setStep("setup")                                                 │
│ dispatch(fetchQrStart())                                         │
│   ↓                                                               │
│ fetch("GET /api/a2f/qrcode", { credentials: "include" })       │
│   ├─ Cookie: token=jwt_token                                     │
│   └─ HTTP GET ────────────────────────────> GET /api/a2f/qrcode │
│                                                   ↓               │
│                                            authMiddleware()      │
│                                               ✅ Token valide     │
│                                                   ↓               │
│                                            qrCode() controller    │
│                                               ↓                   │
│                                            secret = otplib        │
│                                               .generateSecret()   │
│                                               ↓                   │
│                                            QrCodeGenerator        │
│                                               .generate()         │
│                                               ↓                   │
│                                            otplib.generateURI()  │
│                                            qrcode.toDataURL()    │
│                                               ↓                   │
│                                     { image, username, secret }  │
│                                               ↓                   │
│                         200 OK ← ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ←
│ dispatch(fetchQrSuccess({                                        │
│   image: "data:image/png;base64,iVBOR...",                       │
│   secret: "JBSWY3DPEBLW64TMMQ..."                                │
│ }))                                                               │
│   ↓                                                               │
│ [Afficher QR code à l'écran]                                     │
│ [Afficher input pour saisir code OTP]                            │
└──────────────────────────────────────────────────────────────────┘

ÉTAPE 2: Utilisateur scanne le QR code avec son téléphone
┌──────────────────────────────────────────────────────────────────┐
│ Utilisateur                        Google Authenticator          │
│                                                                   │
│ [Scanne QR code avec téléphone]                                  │
│                                     ↓                            │
│                                  Lit la URI:                     │
│   otpauth://totp/EventHub:user@example.com?secret=JBSWY...      │
│                                     ↓                            │
│                                  ✅ Secret enregistré!           │
│                                     ↓                            │
│                                  Génère les codes TOTP           │
│                                  Temps: 0-30s   → 453621         │
│                                  Temps: 30-60s  → 782014         │
│                                  Temps: 60-90s  → 219654         │
│                                     ↓                            │
│                                  Affiche: 453621                 │
└──────────────────────────────────────────────────────────────────┘

ÉTAPE 3: Utilisateur envoie le code OTP
┌──────────────────────────────────────────────────────────────────┐
│ Frontend (React)                   Backend (Node.js)             │
│                                                                   │
│ [Utilisateur tape: 453621]                                       │
│ setOtpToken("453621")                                            │
│   ↓                                                               │
│ [Clique "Confirmer"]                                             │
│ handleConfirmEnable({                                            │
│   otpToken: "453621",                                            │
│   secret: "JBSWY3DPEBLW64TMMQ..."                                │
│ })                                                               │
│   ↓                                                               │
│ dispatch(enableStart())                                          │
│   ↓                                                               │
│ fetch("POST /api/a2f/enable", {                                  │
│   body: { otpToken: "453621", secret: "JBSWY..." }              │
│   credentials: "include"                                         │
│ })                                                               │
│   │                                 ↓                            │
│   │                            authMiddleware()                  │
│   │                               ✅ Token valide               │
│   │                                 ↓                            │
│   ├──────────────────────────→ enableA2F() controller            │
│   │                               ↓                              │
│   │                            EnableA2FUseCase.execute({        │
│   │                              userId: "user123",              │
│   │                              otpToken: "453621",             │
│   │                              secret: "JBSWY..."              │
│   │                            })                                │
│   │                               ↓                              │
│   │                            ⭐ otplib.verify({                │
│   │                              token: "453621",                │
│   │                              secret: "JBSWY..."              │
│   │                            })                                │
│   │                               ↓                              │
│   │                            ✅ OTP VALIDE!                    │
│   │                               ↓                              │
│   │                            userRepository.updateOtp(         │
│   │                              userId,                         │
│   │                              secret,                         │
│   │                              1  ← otpEnabled = 1             │
│   │                            )                                 │
│   │                               ↓                              │
│   │                            generateBackupCodes(6)            │
│   │                            ["ABC12345", "XYZ98765", ...]    │
│   │                               ↓                              │
│   │                            bcrypt.hash() chaque code         │
│   │                               ↓                              │
│   │                            otpBackupCodeRepository.save()    │
│   │                               ↓                              │
│   │             200 OK ← ― ― ― ― ― ― ― ― ― ― ― ― ← 
│   │  {                                                           │
│   │    backupCodes: [                                            │
│   │      "ABC12345",                                             │
│   │      "XYZ98765",                                             │
│   │      "QWE54321",                                             │
│   │      ...                                                     │
│   │    ]                                                         │
│   │  }                                                           │
│                                                                   │
│ dispatch(enableSuccess([...backupCodes]))                        │
│   ↓                                                               │
│ [Afficher codes de backup]                                       │
│ [Message: "A2F activée avec succès!"]                            │
│ ✅ A2F EST MAINTENANT ACTIVÉE!                                    │
└──────────────────────────────────────────────────────────────────┘
```

### 🔄 Scénario 2: Login avec A2F activée

```
┌─────────────────────────────────────────────────────────────────────┐
│                    UTILISATEUR SE CONNECTANT                         │
└─────────────────────────────────────────────────────────────────────┘

ÉTAPE 1: Saisie email/mdp
┌──────────────────────────────────────────────────────────────────┐
│ Frontend (React)                   Backend (Node.js)             │
│                                                                   │
│ [Email: user@example.com]                                        │
│ [Mdp: mypassword]                                                │
│ [Clique Login]                                                   │
│   ↓                                                               │
│ dispatch(LoginUseCase({                                          │
│   email: "user@example.com",                                     │
│   password: "mypassword"                                         │
│ }))                                                              │
│   │                             ↓                                │
│   │                        LoginUseCase.execute()                │
│   │                             ↓                                │
│   │                        userRepository.findByEmail()          │
│   │                        ✅ Utilisateur trouvé                 │
│   │                             ↓                                │
│   │                        bcrypt.compare(mdp)                   │
│   │                        ✅ Mdp correct                        │
│   │                             ↓                                │
│   │                        Vérifier: otpEnabled === 1?           │
│   │                        ✅ OUI! A2F activée!                  │
│   │                             ↓                                │
│   │                        jwt.sign({                            │
│   │                          userId: "user123",                  │
│   │                          email: "user@...",                  │
│   │                          pendingOtp: true    ← KEY!          │
│   │                        }, secret, { expiresIn: "5m" })       │
│   │                             ↓                                │
│   │             200 OK ← ― ― ― ― ― ― ― ― ― ― ←                │
│   │             {                                                │
│   │               requiresOtp: true,                             │
│   │               tempToken: "eyJhbGc..."                        │
│   │             }                                                │
│                                                                   │
│ if (res.requiresOtp) {                                           │
│   // Stocker tempToken                                          │
│   localStorage.setItem("tempToken", res.tempToken)              │
│   // Afficher l'écran de saisie OTP                             │
│   navigateTo("/otp-verification")                               │
│ }                                                                │
└──────────────────────────────────────────────────────────────────┘

ÉTAPE 2: Utilisateur regarde son téléphone et tape le code OTP
┌──────────────────────────────────────────────────────────────────┐
│ Utilisateur                        Google Authenticator          │
│                                                                   │
│ 📱 Ouvre Google Authenticator                                    │
│ 👀 Sees: EventHub  219654                                        │
│ ⏱️  Compte à rebours: 15s avant expiration                        │
│ ✍️ Saisit: 219654                                                │
└──────────────────────────────────────────────────────────────────┘

ÉTAPE 3: Vérifier le code OTP
┌──────────────────────────────────────────────────────────────────┐
│ Frontend (React)                   Backend (Node.js)             │
│                                                                   │
│ [Utilisateur saisi: 219654]                                      │
│ [Clique "Confirmer"]                                             │
│   ↓                                                               │
│ dispatch(VerifyOtpUseCase({                                      │
│   otpToken: "219654",                                            │
│   tempToken: localStorage.getItem("tempToken")                   │
│ }))                                                              │
│   │                             ↓                                │
│   │                        VerifyOtpUseCase.execute()            │
│   │                             ↓                                │
│   │                        jwt.verify(tempToken)                 │
│   │                        ✅ Token valide                       │
│   │                             ↓                                │
│   │                        Vérifier: payload.pendingOtp === true │
│   │                        ✅ OUI, c'est un challenge OTP        │
│   │                             ↓                                │
│   │                        userRepository.findById()             │
│   │                        ✅ Utilisateur trouvé                 │
│   │                             ↓                                │
│   │                        ⭐ otplib.verify({                    │
│   │                          token: "219654",                    │
│   │                          secret: user.otpSecret ← DB         │
│   │                        })                                    │
│   │                             ↓                                │
│   │                        ✅ OTP VALIDE!                        │
│   │                             ↓                                │
│   │                        jwt.sign({                            │
│   │                          userId: "user123",                  │
│   │                          email: "user@...",                  │
│   │                          name: "User Name"                   │
│   │                        }, secret, { expiresIn: "24h" })      │
│   │                             ↓                                │
│   │             200 OK ← ― ― ― ― ― ― ― ― ― ― ←                │
│   │             {                                                │
│   │               token: "eyJhbGc..."  ← TOKEN FINAL             │
│   │               email: "user@...",                             │
│   │               name: "User Name"                              │
│   │             }                                                │
│                                                                   │
│ // ✅ Utilisateur connecté!                                       │
│ localStorage.setItem("token", res.token)                         │
│ navigateTo("/dashboard")                                         │
│ ✅ CONNECTÉ AVEC A2F!                                             │
└──────────────────────────────────────────────────────────────────┘
```

---

## Database

### Schema Prisma - Champs A2F

**Fichier:** `EventHub-Backend/prisma/schema.prisma`

```prisma
model User {
  id              String    @id @default(uuid())
  email           String    @unique
  password        String
  name            String
  role            String    @default("user")
  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt
  
  // 🔐 Champs A2F
  otpSecret       String?              // Secret TOTP stocké en DB
  otpEnabled      Int       @default(0) // 0 = disabled, 1 = enabled
  
  bookings        Booking[]
  organizedEvents Event[]   @relation("Organizer")
  reviews         Review[]
}

// ⚠️ Note: OtpBackupCode n'est pas dans le schema actuellement
// À ajouter si les codes de backup doivent être stockés:
model OtpBackupCode {
  id                    String  @id @default(uuid())
  user_id               String
  codes                 String  // JSON array of bcrypt-hashed codes
  nb_code_used          Int     @default(0)
  nb_consecutive_tests  Int     @default(0)
  createdAt             DateTime @default(now())
}
```

### Flux de données en DB

#### 1. **Activation A2F**
```sql
-- Before Activation
SELECT * FROM "User" WHERE id = 'user123';
┌──────────┬──────────────────┬───────────────┐
│ id       │ otpSecret        │ otpEnabled    │
├──────────┼──────────────────┼───────────────┤
│ user123  │ NULL             │ 0             │
└──────────┴──────────────────┴───────────────┘

-- After Activation
UPDATE "User" 
SET otpSecret = 'JBSWY3DPEBLW64TMMQ...', otpEnabled = 1 
WHERE id = 'user123';

SELECT * FROM "User" WHERE id = 'user123';
┌──────────┬──────────────────┬───────────────┐
│ id       │ otpSecret        │ otpEnabled    │
├──────────┼──────────────────┼───────────────┤
│ user123  │ JBSWY3DPEBLW...  │ 1             │
└──────────┴──────────────────┴───────────────┘

-- Backup codes also saved
INSERT INTO "OtpBackupCode" (user_id, codes, nb_code_used, nb_consecutive_tests)
VALUES (
  'user123',
  '["$2b$10$...", "$2b$10$...", ...]',  ← bcrypt-hashed
  0,
  0
);
```

#### 2. **Vérification lors du Login**
```sql
-- Backend checks during login
SELECT otpEnabled FROM "User" WHERE email = 'user@example.com';
┌─────────────────┐
│ otpEnabled      │
├─────────────────┤
│ 1               │  ← A2F is enabled!
└─────────────────┘

-- If otpEnabled == 1, ask for OTP
-- After correct OTP, generate final token

-- During OTP verification
SELECT otpSecret FROM "User" WHERE id = 'user123';
-- Backend uses this secret to verify the OTP code
```

---

## 📝 Résumé des fichiers et leurs rôles

| Fichier | Rôle | Responsabilités |
|---------|------|-----------------|
| **Backend** | | |
| `a2f.controller.ts` | HTTP Endpoints | qrCode(), enableA2F(), verifyOtp() |
| `EnableA2FUseCase.ts` | Logique métier | Valide OTP, active A2F, génère codes backup |
| `VerifyOtpUseCase.ts` | Logique métier | Vérifie OTP lors du login |
| `LoginUseCase.ts` | Logique métier | Détecte A2F activée, génère tempToken |
| `QrCodeGenerator.ts` | Utilitaires | Génère URI TOTP et image QR |
| `a2fRoutes.ts` | Configuration | Routes publiques/protégées |
| `PrismaUserRepository.ts` | Accès DB | CRUD operations sur User |
| **Frontend** | | |
| `a2f.slice.ts` | État Redux | State management (QR, secret, backupCodes) |
| `a2f.usecase.ts` | Actions Thunk | fetchQrCode(), enableA2F(), disableA2F() |
| `A2FSection.tsx` | Component React | UI pour activation/gestion A2F |
| **Database** | | |
| `schema.prisma` | Schéma | otpSecret, otpEnabled |

---

## 🎯 Points clés à retenir

1. **OTP** = Code à 6 chiffres généré toutes les 30 secondes
2. **A2F** = Système complet de double authentification
3. **Secret** = Clé partagée entre le serveur et le téléphone (stockée en DB)
4. **QR Code** = Encodage du secret et des infos d'authentification
5. **Codes de backup** = Codes de secours hashés en bcrypt
6. **tempToken** = Token JWT temporaire (5 min) pour valider l'OTP
7. **Token final** = JWT des 24h pour accéder à l'app après vérification OTP

---

## 🧪 Cas de test

### Test 1: Activation A2F
- ✅ Se connecter sans A2F
- ✅ Aller à la section A2F
- ✅ Cliquer "Activer A2F"
- ✅ Scanner QR code avec Google Authenticator
- ✅ Taper le code OTP
- ✅ Vérifier que "A2F activée" s'affiche
- ✅ Sauvegarder les codes de backup

### Test 2: Login avec A2F
- ✅ Taper email/mdp
- ✅ Recevoir message "OTP requis"
- ✅ Taper code OTP de Google Authenticator
- ✅ Vérifier connexion réussie

### Test 3: Code OTP invalide
- ✅ Taper code OTP incorrect
- ✅ Vérifier message d'erreur "Code OTP invalide"
- ✅ Pouvoir réessayer

### Test 4: Codes de backup
- ✅ Utiliser un code de backup au lieu d'OTP
- ✅ Vérifier connexion réussie
- ✅ Vérifier que le code ne peut pas être réutilisé
