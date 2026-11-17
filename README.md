import zipfile, pathlib, shutil, textwrap, os
src_zip="/mnt/data/jotachi_karaoke (3) (1).zip"
work=pathlib.Path("/mnt/data/JOTACHI_KARAOKE_ACTUALIZADO")
if work.exists(): shutil.rmtree(work)
work.mkdir()
with zipfile.ZipFile(src_zip) as zz:
    zz.extractall(work)
root=work/"jotachi_karaoke"

# fixed server config from build-time environment
(root/"lib/config").mkdir(parents=True, exist_ok=True)
(root/"lib/config/jotachi_config.dart").write_text(r"""/// URL fija del servidor JOTACHI, definida al compilar la APK.
/// Ejemplo:
/// flutter build apk --release --dart-define=JOTACHI_SERVER_URL=https://api.tudominio.cl
///
/// La clave API NO se incluye en la APK. Se guarda en Ajustes para evitar
/// publicar una credencial privada dentro de la aplicación.
class JotachiConfig {
  static const String serverUrl = String.fromEnvironment(
    'JOTACHI_SERVER_URL',
    defaultValue: '',
  );
}
""", encoding="utf-8")

# Patch AppState: default to compile-time fixed URL, retain saved override.
p=root/"lib/services/app_state.dart"
s=p.read_text()
s=s.replace("import '../models/song.dart';", "import '../models/song.dart';\nimport '../config/jotachi_config.dart';")
s=s.replace("serverUrl = _prefs.getString('serverUrl') ?? '';", "serverUrl = _prefs.getString('serverUrl') ?? JotachiConfig.serverUrl;")
p.write_text(s, encoding="utf-8")

# iTunes catalog search service, official previews only.
(root/"lib/services/music_search_service.dart").write_text(r"""import 'dart:convert';
import 'package:http/http.dart' as http;

/// Busca metadatos musicales en el catálogo público de iTunes.
/// Apple puede entregar una vista previa (normalmente de 30 segundos).
/// No descarga canciones completas ni sustituye una licencia de streaming.
class MusicSearchService {
  static Future<List<MusicSearchResult>> search(
    String query, {
    String country = 'CL',
    int limit = 25,
  }) async {
    final term = query.trim();
    if (term.isEmpty) return [];
    final uri = Uri.https('itunes.apple.com', '/search', {
      'term': term,
      'media': 'music',
      'entity': 'song',
      'country': country,
      'limit': '$limit',
    });
    final response = await http.get(uri, headers: {
      'Accept': 'application/json',
      'User-Agent': 'JotachiKaraoke/1.0',
    }).timeout(const Duration(seconds: 15));
    if (response.statusCode != 200) {
      throw Exception('El buscador respondió ${response.statusCode}.');
    }
    final decoded = jsonDecode(utf8.decode(response.bodyBytes));
    if (decoded is! Map || decoded['results'] is! List) return [];
    return (decoded['results'] as List)
        .whereType<Map>()
        .map((e) => MusicSearchResult.fromJson(Map<String, dynamic>.from(e)))
        .where((e) => e.title.isNotEmpty)
        .toList();
  }
}

class MusicSearchResult {
  const MusicSearchResult({
    required this.title,
    required this.artist,
    required this.album,
    required this.artworkUrl,
    required this.previewUrl,
    required this.trackUrl,
  });

  final String title;
  final String artist;
  final String album;
  final String? artworkUrl;
  final String? previewUrl;
  final String? trackUrl;

  factory MusicSearchResult.fromJson(Map<String, dynamic> json) {
    return MusicSearchResult(
      title: json['trackName'] as String? ?? '',
      artist: json['artistName'] as String? ?? 'Artista desconocido',
      album: json['collectionName'] as String? ?? '',
      artworkUrl: json['artworkUrl100'] as String?,
      previewUrl: json['previewUrl'] as String?,
      trackUrl: json['trackViewUrl'] as String?,
    );
  }
}
""", encoding="utf-8")

# Screen: search catalog and play official previews; external link for full authorized playback.
(root/"lib/screens/music_search_screen.dart").write_text(r"""import 'package:flutter/material.dart';
import 'package:just_audio/just_audio.dart';
import 'package:url_launcher/url_launcher.dart';

import '../services/music_search_service.dart';
import '../theme/app_theme.dart';

class MusicSearchScreen extends StatefulWidget {
  const MusicSearchScreen({super.key});

  @override
  State<MusicSearchScreen> createState() => _MusicSearchScreenState();
}

class _MusicSearchScreenState extends State<MusicSearchScreen> {
  final _query = TextEditingController();
  final _player = AudioPlayer();
  List<MusicSearchResult> _results = [];
  bool _loading = false;
  String? _error;
  String? _playingUrl;

  Future<void> _search() async {
    if (_query.text.trim().isEmpty) return;
    setState(() { _loading = true; _error = null; });
    try {
      final results = await MusicSearchService.search(_query.text);
      if (!mounted) return;
      setState(() => _results = results);
      if (results.isEmpty) setState(() => _error = 'No se encontraron canciones.');
    } catch (e) {
      if (mounted) setState(() => _error = 'No se pudo buscar: $e');
    } finally {
      if (mounted) setState(() => _loading = false);
    }
  }

  Future<void> _preview(MusicSearchResult song) async {
    final url = song.previewUrl;
    if (url == null || url.isEmpty) {
      setState(() => _error = 'Esta canción no tiene vista previa disponible.');
      return;
    }
    try {
      if (_playingUrl == url && _player.playing) {
        await _player.pause();
        if (mounted) setState(() => _playingUrl = null);
        return;
      }
      await _player.setUrl(url);
      await _player.play();
      if (mounted) setState(() => _playingUrl = url);
    } catch (e) {
      if (mounted) setState(() => _error = 'No se pudo reproducir la vista previa.');
    }
  }

  Future<void> _openTrack(MusicSearchResult song) async {
    final url = song.trackUrl;
    if (url == null) return;
    final uri = Uri.tryParse(url);
    if (uri != null) await launchUrl(uri, mode: LaunchMode.externalApplication);
  }

  @override
  void dispose() {
    _query.dispose();
    _player.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) => Scaffold(
    appBar: AppBar(title: const Text('Buscar música')),
    body: Column(children: [
      Padding(
        padding: const EdgeInsets.all(16),
        child: Row(children: [
          Expanded(child: TextField(
            controller: _query,
            textInputAction: TextInputAction.search,
            onSubmitted: (_) => _search(),
            decoration: const InputDecoration(
              hintText: 'Canción o artista',
              prefixIcon: Icon(Icons.search),
              border: OutlineInputBorder(),
            ),
          )),
          const SizedBox(width: 8),
          IconButton.filled(
            onPressed: _loading ? null : _search,
            icon: const Icon(Icons.search),
          ),
        ]),
      ),
      if (_loading) const LinearProgressIndicator(color: JColors.blue),
      if (_error != null)
        Padding(
          padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 8),
          child: Text(_error!, style: const TextStyle(color: Colors.orangeAccent)),
        ),
      const Padding(
        padding: EdgeInsets.symmetric(horizontal: 16),
        child: Text(
          'Resultados del catálogo musical. Las vistas previas dependen de cada canción; para escuchar completa, abre el servicio autorizado.',
          style: TextStyle(color: JColors.textDim, fontSize: 12),
        ),
      ),
      Expanded(child: ListView.builder(
        itemCount: _results.length,
        itemBuilder: (_, i) {
          final song = _results[i];
          final isPlaying = _playingUrl == song.previewUrl && _player.playing;
          return ListTile(
            leading: song.artworkUrl == null
                ? const CircleAvatar(child: Icon(Icons.music_note))
                : ClipRRect(
                    borderRadius: BorderRadius.circular(6),
                    child: Image.network(song.artworkUrl!, width: 52, height: 52, fit: BoxFit.cover,
                      errorBuilder: (_, __, ___) => const Icon(Icons.music_note)),
                  ),
            title: Text(song.title, maxLines: 1, overflow: TextOverflow.ellipsis),
            subtitle: Text('${song.artist}${song.album.isEmpty ? '' : ' • ${song.album}'}',
              maxLines: 2, overflow: TextOverflow.ellipsis),
            trailing: Wrap(spacing: 0, children: [
              IconButton(
                tooltip: 'Vista previa',
                icon: Icon(isPlaying ? Icons.pause_circle : Icons.play_circle,
                  color: JColors.blue, size: 30),
                onPressed: () => _preview(song),
              ),
              IconButton(
                tooltip: 'Abrir servicio musical',
                icon: const Icon(Icons.open_in_new, color: JColors.gold),
                onPressed: song.trackUrl == null ? null : () => _openTrack(song),
              ),
            ]),
          );
        },
      )),
    ]),
  );
}
""", encoding="utf-8")

# Add url_launcher dependency
p=root/"pubspec.yaml"
s=p.read_text()
s=s.replace("  http: ^1.2.2\n", "  http: ^1.2.2\n  url_launcher: ^6.3.1\n")
p.write_text(s, encoding="utf-8")

# Add search action to existing home screen
p=root/"lib/screens/home_screen.dart"
s=p.read_text()
s=s.replace("import 'player_screen.dart';", "import 'player_screen.dart';\nimport 'music_search_screen.dart';")
needle="""        actions: [
          IconButton(
            icon: const Icon(Icons.settings, color: JColors.blue),"""
repl="""        actions: [
          IconButton(
            tooltip: 'Buscar música por internet',
            icon: const Icon(Icons.search, color: JColors.gold),
            onPressed: () => Navigator.push(context,
                MaterialPageRoute(builder: (_) => const MusicSearchScreen())),
          ),
          IconButton(
            icon: const Icon(Icons.settings, color: JColors.blue),"""
s=s.replace(needle,repl)
p.write_text(s, encoding="utf-8")

# Auto-lyrics fallback: if LRCLIB has no synchronized lyrics, attempt Whisper on the configured server.
p=root/"lib/screens/player_screen.dart"
s=p.read_text()
old="""      final ok = await state.fetchLyricsOnline(song, durationSec: secs);
      if (ok) await _loadLyrics();
    } catch (_) {
      // sin conexión: el usuario puede usar el menú ⋮
    }"""
new="""      final ok = await state.fetchLyricsOnline(song, durationSec: secs);
      if (ok) {
        await _loadLyrics();
      } else if (state.serverUrl.isNotEmpty) {
        // Fallback automático: transcribir con Whisper en el servidor fijo.
        final generated = await state.generateLyricsAI(song);
        if (generated != null && generated.isNotEmpty) await _loadLyrics();
      }
    } catch (_) {
      // Si no hay conexión o el servidor no responde, el usuario puede reintentar.
    }"""
if old not in s:
    print("WARNING auto lyrics block not found")
else:
    s=s.replace(old,new)
p.write_text(s, encoding="utf-8")

# Update settings copy to explain fixed server via build config
p=root/"lib/screens/settings_screen.dart"
s=p.read_text()
oldtxt="""              'La separación local (sin internet) aún no está implementada. '
              'La app usa un servidor con IA para quitar la voz y generar letras: '
              'abre el cuaderno JOTACHI_Servidor en Google Colab, ejecútalo y '
              'pega aquí la URL y la clave que muestre al final.',"""
newtxt="""              'La app usa un servidor permanente con IA para separar voz y generar letras. '
              'La URL puede venir integrada al compilar la APK; si el administrador '
              'te entrega una clave, guárdala aquí una sola vez. No uses una URL temporal '
              'de Google Colab para un servidor permanente.',"""
s=s.replace(oldtxt,newtxt)
s=s.replace("hintText: 'https://xxxx.trycloudflare.com',", "hintText: 'https://api.tudominio.cl',")
p.write_text(s, encoding="utf-8")

# Docker deployment files
srv=root/"server"
(srv/"Dockerfile").write_text("""FROM python:3.11-slim
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1 JOBS_DIR=/data/jobs WHISPER_MODEL=small
RUN apt-get update && apt-get install -y --no-install-recommends ffmpeg libsndfile1 && rm -rf /var/lib/apt/lists/*
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY main.py lyrics_utils.py ./
RUN mkdir -p /data/jobs
EXPOSE 8000
CMD [\\"uvicorn\\", \\"main:app\\", \\"--host\\", \\"0.0.0.0\\", \\"--port\\", \\"8000\\"]
""", encoding="utf-8")
(srv/"docker-compose.yml").write_text("""services:
  jotachi-ai:
    build: .
    container_name: jotachi-ai
    restart: unless-stopped
    ports:
      - \\"8000:8000\\"
    environment:
      API_KEY: ${API_KEY}
      WHISPER_MODEL: ${WHISPER_MODEL:-small}
      JOBS_DIR: /data/jobs
    volumes:
      - jotachi_jobs:/data/jobs
volumes:
  jotachi_jobs:
""", encoding="utf-8")
(srv/".env.example").write_text("API_KEY=REEMPLAZA_POR_UNA_CLAVE_LARGA_Y_PRIVADA\nWHISPER_MODEL=small\n", encoding="utf-8")

# Add deployment instructions and honest feature limitations.
(root/"INSTALACION_SERVIDOR_FIJO_Y_BUSCADOR_ES.txt").write_text("""JOTACHI KARAOKE — ACTUALIZACIÓN
================================

QUÉ SE INTEGRÓ EN EL PROYECTO
- URL del servidor IA configurable al compilar (JOTACHI_SERVER_URL).
- Se conserva el campo de clave API en Ajustes.
- Buscador de canciones del catálogo iTunes en la pantalla principal.
- Reproducción de vistas previas oficiales cuando están disponibles.
- Enlace para abrir la canción en el servicio musical.
- Búsqueda automática de letra sincronizada en LRCLIB y, si no se encuentra,
  intento de transcripción Whisper en el servidor configurado.
- Archivos Docker para alojar el backend en un servidor permanente.

IMPORTANTE SOBRE EL BUSCADOR
El catálogo devuelve metadatos y algunas vistas previas, no el archivo completo
de todas las canciones. La reproducción completa debe hacerse mediante un
proveedor autorizado (por ejemplo, el servicio musical que tenga licencia).
No se descarga música protegida ni se evita el pago o DRM de otros servicios.

INSTALAR EL SERVIDOR UNA SOLA VEZ
Se necesita un VPS Linux que permanezca encendido, Docker, dominio y HTTPS.
Para Demucs/Whisper se recomienda servidor con GPU NVIDIA compatible; en CPU
puede ser muy lento y consumir bastante memoria.

1. Copia la carpeta server/ al VPS.
2. En server ejecuta: cp .env.example .env
3. Edita .env y reemplaza API_KEY por una clave privada larga.
4. Ejecuta: docker compose up -d --build
5. Configura HTTPS con un proxy inverso (Caddy/Nginx) hacia el puerto 8000.
6. Comprueba https://TU-DOMINIO/health usando la clave API.
7. Conserva ese dominio; será la URL única para procesar todas tus canciones.

SEGURIDAD
No se incluye la clave API privada dentro del APK. En esta versión se introduce
una vez en Ajustes y queda guardada en el teléfono. Para una app pública o muchos
usuarios, se recomienda autenticación por usuario y límites de consumo.

COMPILAR APK CON URL FIJA
En el proyecto:
flutter pub get
flutter build apk --release --dart-define=JOTACHI_SERVER_URL=https://TU-DOMINIO

La URL queda integrada en la aplicación. La clave se guarda una vez en Ajustes.

ARCHIVO DE GITHUB
El workflow actual del repositorio compila el archivo jotachi_karaoke.zip.
Para actualizarlo, reemplaza ese ZIP en la raíz del repositorio por el ZIP
jotachi_karaoke.zip incluido en este paquete, conservando el nombre exacto.
Después ejecuta GitHub Actions. El nuevo APK se publica como artefacto.

ESTADO Y LIMITACIONES
- El buscador está integrado y muestra resultados/previews de iTunes.
- No hay descarga de canciones completas desde iTunes.
- El servidor se entrega preparado, pero no está alojado ni tiene dominio público:
  hay que desplegarlo en un VPS y configurar HTTPS.
- La transcripción Whisper puede equivocarse, especialmente con coros, ruido
  o canciones con voces superpuestas.
- La separación de voz requiere conexión a internet y recursos de servidor.
""", encoding="utf-8")

# Add server dependency for ffmpeg already installed by Docker; ensure pip req has all needed packages.
# Existing requirements inspected and preserved.

# Build the source zip in the exact expected top-level directory.
inner_zip="/mnt/data/jotachi_karaoke.zip"
with zipfile.ZipFile(inner_zip,"w",zipfile.ZIP_DEFLATED) as zz:
    for f in root.rglob("*"):
        if f.is_file():
            zz.write(f, f.relative_to(work))

# Outer zip for convenient transfer, containing expected archive and docs.
outer="/mnt/data/JOTACHI_KARAOKE_PROYECTO_ACTUALIZADO.zip"
with zipfile.ZipFile(outer,"w",zipfile.ZIP_DEFLATED) as zz:
    zz.write(inner_zip,"jotachi_karaoke.zip")
    zz.write(root/"INSTALACION_SERVIDOR_FIJO_Y_BUSCADOR_ES.txt","INSTALACION_SERVIDOR_FIJO_Y_BUSCADOR_ES.txt")
print("Created", inner_zip, os.path.getsize(inner_zip), outer, os.path.getsize(outer))
import zipfile, pathlib, shutil, textwrap, os
src_zip="/mnt/data/jotachi_karaoke (3) (1).zip"
work=pathlib.Path("/mnt/data/JOTACHI_KARAOKE_ACTUALIZADO")
if work.exists(): shutil.rmtree(work)
work.mkdir()
with zipfile.ZipFile(src_zip) as zz:
    zz.extractall(work)
root=work/"jotachi_karaoke"

# fixed server config from build-time environment
(root/"lib/config").mkdir(parents=True, exist_ok=True)
(root/"lib/config/jotachi_config.dart").write_text(r"""/// URL fija del servidor JOTACHI, definida al compilar la APK.
/// Ejemplo:
/// flutter build apk --release --dart-define=JOTACHI_SERVER_URL=https://api.tudominio.cl
///
/// La clave API NO se incluye en la APK. Se guarda en Ajustes para evitar
/// publicar una credencial privada dentro de la aplicación.
class JotachiConfig {
  static const String serverUrl = String.fromEnvironment(
    'JOTACHI_SERVER_URL',
    defaultValue: '',
  );
}
""", encoding="utf-8")

# Patch AppState: default to compile-time fixed URL, retain saved override.
p=root/"lib/services/app_state.dart"
s=p.read_text()
s=s.replace("import '../models/song.dart';", "import '../models/song.dart';\nimport '../config/jotachi_config.dart';")
s=s.replace("serverUrl = _prefs.getString('serverUrl') ?? '';", "serverUrl = _prefs.getString('serverUrl') ?? JotachiConfig.serverUrl;")
p.write_text(s, encoding="utf-8")

# iTunes catalog search service, official previews only.
(root/"lib/services/music_search_service.dart").write_text(r"""import 'dart:convert';
import 'package:http/http.dart' as http;

/// Busca metadatos musicales en el catálogo público de iTunes.
/// Apple puede entregar una vista previa (normalmente de 30 segundos).
/// No descarga canciones completas ni sustituye una licencia de streaming.
class MusicSearchService {
  static Future<List<MusicSearchResult>> search(
    String query, {
    String country = 'CL',
    int limit = 25,
  }) async {
    final term = query.trim();
    if (term.isEmpty) return [];
    final uri = Uri.https('itunes.apple.com', '/search', {
      'term': term,
      'media': 'music',
      'entity': 'song',
      'country': country,
      'limit': '$limit',
    });
    final response = await http.get(uri, headers: {
      'Accept': 'application/json',
      'User-Agent': 'JotachiKaraoke/1.0',
    }).timeout(const Duration(seconds: 15));
    if (response.statusCode != 200) {
      throw Exception('El buscador respondió ${response.statusCode}.');
    }
    final decoded = jsonDecode(utf8.decode(response.bodyBytes));
    if (decoded is! Map || decoded['results'] is! List) return [];
    return (decoded['results'] as List)
        .whereType<Map>()
        .map((e) => MusicSearchResult.fromJson(Map<String, dynamic>.from(e)))
        .where((e) => e.title.isNotEmpty)
        .toList();
  }
}

class MusicSearchResult {
  const MusicSearchResult({
    required this.title,
    required this.artist,
    required this.album,
    required this.artworkUrl,
    required this.previewUrl,
    required this.trackUrl,
  });

  final String title;
  final String artist;
  final String album;
  final String? artworkUrl;
  final String? previewUrl;
  final String? trackUrl;

  factory MusicSearchResult.fromJson(Map<String, dynamic> json) {
    return MusicSearchResult(
      title: json['trackName'] as String? ?? '',
      artist: json['artistName'] as String? ?? 'Artista desconocido',
      album: json['collectionName'] as String? ?? '',
      artworkUrl: json['artworkUrl100'] as String?,
      previewUrl: json['previewUrl'] as String?,
      trackUrl: json['trackViewUrl'] as String?,
    );
  }
}
""", encoding="utf-8")

# Screen: search catalog and play official previews; external link for full authorized playback.
(root/"lib/screens/music_search_screen.dart").write_text(r"""import 'package:flutter/material.dart';
import 'package:just_audio/just_audio.dart';
import 'package:url_launcher/url_launcher.dart';

import '../services/music_search_service.dart';
import '../theme/app_theme.dart';

class MusicSearchScreen extends StatefulWidget {
  const MusicSearchScreen({super.key});

  @override
  State<MusicSearchScreen> createState() => _MusicSearchScreenState();
}

class _MusicSearchScreenState extends State<MusicSearchScreen> {
  final _query = TextEditingController();
  final _player = AudioPlayer();
  List<MusicSearchResult> _results = [];
  bool _loading = false;
  String? _error;
  String? _playingUrl;

  Future<void> _search() async {
    if (_query.text.trim().isEmpty) return;
    setState(() { _loading = true; _error = null; });
    try {
      final results = await MusicSearchService.search(_query.text);
      if (!mounted) return;
      setState(() => _results = results);
      if (results.isEmpty) setState(() => _error = 'No se encontraron canciones.');
    } catch (e) {
      if (mounted) setState(() => _error = 'No se pudo buscar: $e');
    } finally {
      if (mounted) setState(() => _loading = false);
    }
  }

  Future<void> _preview(MusicSearchResult song) async {
    final url = song.previewUrl;
    if (url == null || url.isEmpty) {
      setState(() => _error = 'Esta canción no tiene vista previa disponible.');
      return;
    }
    try {
      if (_playingUrl == url && _player.playing) {
        await _player.pause();
        if (mounted) setState(() => _playingUrl = null);
        return;
      }
      await _player.setUrl(url);
      await _player.play();
      if (mounted) setState(() => _playingUrl = url);
    } catch (e) {
      if (mounted) setState(() => _error = 'No se pudo reproducir la vista previa.');
    }
  }

  Future<void> _openTrack(MusicSearchResult song) async {
    final url = song.trackUrl;
    if (url == null) return;
    final uri = Uri.tryParse(url);
    if (uri != null) await launchUrl(uri, mode: LaunchMode.externalApplication);
  }

  @override
  void dispose() {
    _query.dispose();
    _player.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) => Scaffold(
    appBar: AppBar(title: const Text('Buscar música')),
    body: Column(children: [
      Padding(
        padding: const EdgeInsets.all(16),
        child: Row(children: [
          Expanded(child: TextField(
            controller: _query,
            textInputAction: TextInputAction.search,
            onSubmitted: (_) => _search(),
            decoration: const InputDecoration(
              hintText: 'Canción o artista',
              prefixIcon: Icon(Icons.search),
              border: OutlineInputBorder(),
            ),
          )),
          const SizedBox(width: 8),
          IconButton.filled(
            onPressed: _loading ? null : _search,
            icon: const Icon(Icons.search),
          ),
        ]),
      ),
      if (_loading) const LinearProgressIndicator(color: JColors.blue),
      if (_error != null)
        Padding(
          padding: const EdgeInsets.symmetric(horizontal: 16, vertical: 8),
          child: Text(_error!, style: const TextStyle(color: Colors.orangeAccent)),
        ),
      const Padding(
        padding: EdgeInsets.symmetric(horizontal: 16),
        child: Text(
          'Resultados del catálogo musical. Las vistas previas dependen de cada canción; para escuchar completa, abre el servicio autorizado.',
          style: TextStyle(color: JColors.textDim, fontSize: 12),
        ),
      ),
      Expanded(child: ListView.builder(
        itemCount: _results.length,
        itemBuilder: (_, i) {
          final song = _results[i];
          final isPlaying = _playingUrl == song.previewUrl && _player.playing;
          return ListTile(
            leading: song.artworkUrl == null
                ? const CircleAvatar(child: Icon(Icons.music_note))
                : ClipRRect(
                    borderRadius: BorderRadius.circular(6),
                    child: Image.network(song.artworkUrl!, width: 52, height: 52, fit: BoxFit.cover,
                      errorBuilder: (_, __, ___) => const Icon(Icons.music_note)),
                  ),
            title: Text(song.title, maxLines: 1, overflow: TextOverflow.ellipsis),
            subtitle: Text('${song.artist}${song.album.isEmpty ? '' : ' • ${song.album}'}',
              maxLines: 2, overflow: TextOverflow.ellipsis),
            trailing: Wrap(spacing: 0, children: [
              IconButton(
                tooltip: 'Vista previa',
                icon: Icon(isPlaying ? Icons.pause_circle : Icons.play_circle,
                  color: JColors.blue, size: 30),
                onPressed: () => _preview(song),
              ),
              IconButton(
                tooltip: 'Abrir servicio musical',
                icon: const Icon(Icons.open_in_new, color: JColors.gold),
                onPressed: song.trackUrl == null ? null : () => _openTrack(song),
              ),
            ]),
          );
        },
      )),
    ]),
  );
}
""", encoding="utf-8")

# Add url_launcher dependency
p=root/"pubspec.yaml"
s=p.read_text()
s=s.replace("  http: ^1.2.2\n", "  http: ^1.2.2\n  url_launcher: ^6.3.1\n")
p.write_text(s, encoding="utf-8")

# Add search action to existing home screen
p=root/"lib/screens/home_screen.dart"
s=p.read_text()
s=s.replace("import 'player_screen.dart';", "import 'player_screen.dart';\nimport 'music_search_screen.dart';")
needle="""        actions: [
          IconButton(
            icon: const Icon(Icons.settings, color: JColors.blue),"""
repl="""        actions: [
          IconButton(
            tooltip: 'Buscar música por internet',
            icon: const Icon(Icons.search, color: JColors.gold),
            onPressed: () => Navigator.push(context,
                MaterialPageRoute(builder: (_) => const MusicSearchScreen())),
          ),
          IconButton(
            icon: const Icon(Icons.settings, color: JColors.blue),"""
s=s.replace(needle,repl)
p.write_text(s, encoding="utf-8")

# Auto-lyrics fallback: if LRCLIB has no synchronized lyrics, attempt Whisper on the configured server.
p=root/"lib/screens/player_screen.dart"
s=p.read_text()
old="""      final ok = await state.fetchLyricsOnline(song, durationSec: secs);
      if (ok) await _loadLyrics();
    } catch (_) {
      // sin conexión: el usuario puede usar el menú ⋮
    }"""
new="""      final ok = await state.fetchLyricsOnline(song, durationSec: secs);
      if (ok) {
        await _loadLyrics();
      } else if (state.serverUrl.isNotEmpty) {
        // Fallback automático: transcribir con Whisper en el servidor fijo.
        final generated = await state.generateLyricsAI(song);
        if (generated != null && generated.isNotEmpty) await _loadLyrics();
      }
    } catch (_) {
      // Si no hay conexión o el servidor no responde, el usuario puede reintentar.
    }"""
if old not in s:
    print("WARNING auto lyrics block not found")
else:
    s=s.replace(old,new)
p.write_text(s, encoding="utf-8")

# Update settings copy to explain fixed server via build config
p=root/"lib/screens/settings_screen.dart"
s=p.read_text()
oldtxt="""              'La separación local (sin internet) aún no está implementada. '
              'La app usa un servidor con IA para quitar la voz y generar letras: '
              'abre el cuaderno JOTACHI_Servidor en Google Colab, ejecútalo y '
              'pega aquí la URL y la clave que muestre al final.',"""
newtxt="""              'La app usa un servidor permanente con IA para separar voz y generar letras. '
              'La URL puede venir integrada al compilar la APK; si el administrador '
              'te entrega una clave, guárdala aquí una sola vez. No uses una URL temporal '
              'de Google Colab para un servidor permanente.',"""
s=s.replace(oldtxt,newtxt)
s=s.replace("hintText: 'https://xxxx.trycloudflare.com',", "hintText: 'https://api.tudominio.cl',")
p.write_text(s, encoding="utf-8")

# Docker deployment files
srv=root/"server"
(srv/"Dockerfile").write_text("""FROM python:3.11-slim
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1 JOBS_DIR=/data/jobs WHISPER_MODEL=small
RUN apt-get update && apt-get install -y --no-install-recommends ffmpeg libsndfile1 && rm -rf /var/lib/apt/lists/*
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY main.py lyrics_utils.py ./
RUN mkdir -p /data/jobs
EXPOSE 8000
CMD [\\"uvicorn\\", \\"main:app\\", \\"--host\\", \\"0.0.0.0\\", \\"--port\\", \\"8000\\"]
""", encoding="utf-8")
(srv/"docker-compose.yml").write_text("""services:
  jotachi-ai:
    build: .
    container_name: jotachi-ai
    restart: unless-stopped
    ports:
      - \\"8000:8000\\"
    environment:
      API_KEY: ${API_KEY}
      WHISPER_MODEL: ${WHISPER_MODEL:-small}
      JOBS_DIR: /data/jobs
    volumes:
      - jotachi_jobs:/data/jobs
volumes:
  jotachi_jobs:
""", encoding="utf-8")
(srv/".env.example").write_text("API_KEY=REEMPLAZA_POR_UNA_CLAVE_LARGA_Y_PRIVADA\nWHISPER_MODEL=small\n", encoding="utf-8")

# Add deployment instructions and honest feature limitations.
(root/"INSTALACION_SERVIDOR_FIJO_Y_BUSCADOR_ES.txt").write_text("""JOTACHI KARAOKE — ACTUALIZACIÓN
================================

QUÉ SE INTEGRÓ EN EL PROYECTO
- URL del servidor IA configurable al compilar (JOTACHI_SERVER_URL).
- Se conserva el campo de clave API en Ajustes.
- Buscador de canciones del catálogo iTunes en la pantalla principal.
- Reproducción de vistas previas oficiales cuando están disponibles.
- Enlace para abrir la canción en el servicio musical.
- Búsqueda automática de letra sincronizada en LRCLIB y, si no se encuentra,
  intento de transcripción Whisper en el servidor configurado.
- Archivos Docker para alojar el backend en un servidor permanente.

IMPORTANTE SOBRE EL BUSCADOR
El catálogo devuelve metadatos y algunas vistas previas, no el archivo completo
de todas las canciones. La reproducción completa debe hacerse mediante un
proveedor autorizado (por ejemplo, el servicio musical que tenga licencia).
No se descarga música protegida ni se evita el pago o DRM de otros servicios.

INSTALAR EL SERVIDOR UNA SOLA VEZ
Se necesita un VPS Linux que permanezca encendido, Docker, dominio y HTTPS.
Para Demucs/Whisper se recomienda servidor con GPU NVIDIA compatible; en CPU
puede ser muy lento y consumir bastante memoria.

1. Copia la carpeta server/ al VPS.
2. En server ejecuta: cp .env.example .env
3. Edita .env y reemplaza API_KEY por una clave privada larga.
4. Ejecuta: docker compose up -d --build
5. Configura HTTPS con un proxy inverso (Caddy/Nginx) hacia el puerto 8000.
6. Comprueba https://TU-DOMINIO/health usando la clave API.
7. Conserva ese dominio; será la URL única para procesar todas tus canciones.

SEGURIDAD
No se incluye la clave API privada dentro del APK. En esta versión se introduce
una vez en Ajustes y queda guardada en el teléfono. Para una app pública o muchos
usuarios, se recomienda autenticación por usuario y límites de consumo.

COMPILAR APK CON URL FIJA
En el proyecto:
flutter pub get
flutter build apk --release --dart-define=JOTACHI_SERVER_URL=https://TU-DOMINIO

La URL queda integrada en la aplicación. La clave se guarda una vez en Ajustes.


ARCHIVO DE GITHUB
El workflow actual del repositorio compila el archivo jotachi_karaoke.zip.
Para actualizarlo, reemplaza ese ZIP en la raíz del repositorio por el ZIP
jotachi_karaoke.zip incluido en este paquete, conservando el nombre exacto.
Después ejecuta GitHub Actions. El nuevo APK se publica como artefacto.

ESTADO Y LIMITACIONES
- El buscador está integrado y muestra resultados/previews de iTunes.
- No hay descarga de canciones completas desde iTunes.
- El servidor se entrega preparado, pero no está alojado ni tiene dominio público:
  hay que desplegarlo en un VPS y configurar HTTPS.
- La transcripción Whisper puede equivocarse, especialmente con coros, ruido
  o canciones con voces superpuestas.
- La separación de voz requiere conexión a internet y recursos de servidor.
""", encoding="utf-8")

# Add server dependency for ffmpeg already installed by Docker; ensure pip req has all needed packages.
# Existing requirements inspected and preserved.

# Build the source zip in the exact expected top-level directory.
inner_zip="/mnt/data/jotachi_karaoke.zip"
with zipfile.ZipFile(inner_zip,"w",zipfile.ZIP_DEFLATED) as zz:
    for f in root.rglob("*"):
        if f.is_file():
            zz.write(f, f.relative_to(work))

# Outer zip for convenient transfer, containing expected archive and docs.
outer="/mnt/data/JOTACHI_KARAOKE_PROYECTO_ACTUALIZADO.zip"
with zipfile.ZipFile(outer,"w",zipfile.ZIP_DEFLATED) as zz:
    zz.write(inner_zip,"jotachi_karaoke.zip")
    zz.write(root/"INSTALACION_SERVIDOR_FIJO_Y_BUSCADOR_ES.txt","INSTALACION_SERVIDOR_FIJO_Y_BUSCADOR_ES.txt")
print("Created", inner_zip, os.path.getsize(inner_zip), outer, os.path.getsize(outer))
