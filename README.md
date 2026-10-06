# AspModder1
// aspmodder1 - Play Store style Mod Store (Flutter)
// Data source: Blogger RSS feed of asplovevlog1.blogspot.com
//
// Blogger post rules (so tabs work automatically):
//   - Label "Games"    -> shows in the Games tab
//   - Label "Apps"     -> shows in the Apps tab
//   - Label "Featured" -> shows in the top banner (if none, latest 5 posts)
//   - Any other label  -> shows in the Categories tab
//   - Download button  -> first link in the post whose text/URL contains
//                         "download", otherwise the post page itself

import 'dart:async';
import 'dart:convert';

import 'package:cached_network_image/cached_network_image.dart';
import 'package:flutter/material.dart';
import 'package:http/http.dart' as http;
import 'package:url_launcher/url_launcher.dart';
import 'package:xml/xml.dart';

const String kFeedUrl =
    'https://asplovevlog1.blogspot.com/feeds/posts/default?alt=rss&max-results=150';
const Color kBrand = Color(0xFF01875F); // Play Store green

void main() => runApp(const AspModderApp());

// ───────────────────────── App ─────────────────────────

class AspModderApp extends StatelessWidget {
  const AspModderApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'aspmodder1',
      debugShowCheckedModeBanner: false,
      themeMode: ThemeMode.system,
      theme: ThemeData(
        useMaterial3: true,
        colorScheme: ColorScheme.fromSeed(seedColor: kBrand),
      ),
      darkTheme: ThemeData(
        useMaterial3: true,
        colorScheme: ColorScheme.fromSeed(
          seedColor: kBrand,
          brightness: Brightness.dark,
        ),
      ),
      home: const HomePage(),
    );
  }
}

// ───────────────────────── Model + RSS ─────────────────────────

class ModApp {
  ModApp({
    required this.title,
    required this.link,
    required this.download,
    required this.icon,
    required this.cover,
    required this.labels,
  });

  final String title, link, download, icon, cover;
  final List<String> labels;

  bool has(List<String> keys) =>
      labels.any((l) => keys.contains(l.toLowerCase()));
}

String _upsize(String url) =>
    url.replaceAllMapped(RegExp(r'/s\d+(-c)?/'), (_) => '/s400/');

Future<List<ModApp>> fetchApps() async {
  final res =
      await http.get(Uri.parse(kFeedUrl)).timeout(const Duration(seconds: 20));
  if (res.statusCode != 200) {
    throw Exception('Feed error (${res.statusCode})');
  }
  final doc = XmlDocument.parse(utf8.decode(res.bodyBytes));
  return doc.findAllElements('item').map(_parseItem).toList();
}

ModApp _parseItem(XmlElement e) {
  String text(String n) => e.getElement(n)?.innerText.trim() ?? '';
  final html = text('description');
  final link = text('link');

  // First image inside the post body
  final bodyImg = RegExp(r'''<img[^>]+src=["']([^"']+)["']''',
              caseSensitive: false)
          .firstMatch(html)
          ?.group(1) ??
      '';

  // <media:thumbnail url="..."/> (Blogger adds this when a post has an image)
  var thumb = '';
  for (final n in e.descendants.whereType<XmlElement>()) {
    if (n.name.local == 'thumbnail') {
      thumb = n.getAttribute('url') ?? '';
      break;
    }
  }

  // Download link: first <a> that looks like a download
  var download = link;
  final anchors = RegExp(r'''<a[^>]+href=["']([^"']+)["'][^>]*>(.*?)</a>''',
      caseSensitive: false, dotAll: true);
  for (final m in anchors.allMatches(html)) {
    final href = m.group(1)!.replaceAll('&amp;', '&');
    if (m.group(2)!.toLowerCase().contains('download') ||
        href.toLowerCase().contains('download')) {
      download = href;
      break;
    }
  }

  final icon = thumb.isNotEmpty ? _upsize(thumb) : bodyImg;
  return ModApp(
    title: text('title'),
    link: link,
    download: download,
    icon: icon,
    cover: bodyImg.isNotEmpty ? bodyImg : icon,
    labels: e
        .findElements('category')
        .map((c) => c.innerText.trim())
        .where((s) => s.isNotEmpty)
        .toList(),
  );
}

Future<void> openUrl(BuildContext context, String url,
    {bool inApp = false}) async {
  try {
    final ok = await launchUrl(
      Uri.parse(url),
      mode: inApp
          ? LaunchMode.inAppBrowserView
          : LaunchMode.externalApplication,
    );
    if (!ok) throw Exception('cannot launch');
  } catch (_) {
    if (context.mounted) {
      ScaffoldMessenger.of(context).showSnackBar(
        const SnackBar(content: Text('Link open nahi ho paaya')),
      );
    }
  }
}

// ───────────────────────── Home ─────────────────────────

class HomePage extends StatefulWidget {
  const HomePage({super.key});

  @override
  State<HomePage> createState() => _HomePageState();
}

class _HomePageState extends State<HomePage>
    with SingleTickerProviderStateMixin {
  late final TabController _tabs = TabController(length: 4, vsync: this);
  final TextEditingController _search = TextEditingController();
  List<ModApp> _all = [];
  bool _loading = true;
  String? _error;

  @override
  void initState() {
    super.initState();
    _load();
  }

  @override
  void dispose() {
    _tabs.dispose();
    _search.dispose();
    super.dispose();
  }

  Future<void> _load() async {
    setState(() {
      _loading = true;
      _error = null;
    });
    try {
      final apps = await fetchApps();
      if (!mounted) return;
      setState(() => _all = apps);
    } catch (_) {
      if (!mounted) return;
      setState(() => _error = 'Feed load nahi hui. Internet check karke retry karein.');
    } finally {
      if (mounted) setState(() => _loading = false);
    }
  }

  String get _q => _search.text.trim().toLowerCase();

  List<ModApp> _filter(Iterable<ModApp> list) => list
      .where((a) => _q.isEmpty || a.title.toLowerCase().contains(_q))
      .toList();

  @override
  Widget build(BuildContext context) {
    final featured = _all.where((a) => a.has(['featured'])).toList();
    final banner = (featured.isNotEmpty ? featured : _all).take(5).toList();

    Widget body;
    if (_loading) {
      body = const Center(child: CircularProgressIndicator());
    } else if (_error != null) {
      body = Center(
        child: Padding(
          padding: const EdgeInsets.all(24),
          child: Column(
            mainAxisSize: MainAxisSize.min,
            children: [
              const Icon(Icons.wifi_off, size: 48),
              const SizedBox(height: 12),
              Text(_error!, textAlign: TextAlign.center),
              const SizedBox(height: 16),
              FilledButton(onPressed: _load, child: const Text('Retry')),
            ],
          ),
        ),
      );
    } else {
      body = TabBarView(
        controller: _tabs,
        children: [
          AppGrid(
            apps: _filter(_all),
            banner: _q.isEmpty ? banner : const [],
            onRefresh: _load,
          ),
          AppGrid(
            apps: _filter(_all.where((a) => a.has(['games', 'game']))),
            onRefresh: _load,
          ),
          AppGrid(
            apps: _filter(_all.where((a) => a.has(['apps', 'app']))),
            onRefresh: _load,
          ),
          CategoriesTab(apps: _all, query: _q, onRefresh: _load),
        ],
      );
    }

    return Scaffold(
      body: SafeArea(
        child: Column(
          children: [
            Padding(
              padding: const EdgeInsets.fromLTRB(16, 12, 16, 8),
              child: SearchBar(
                controller: _search,
                hintText: 'Search mods on aspmodder1',
                leading: const Icon(Icons.search),
                trailing: [
                  if (_search.text.isNotEmpty)
                    IconButton(
                      icon: const Icon(Icons.close),
                      onPressed: () {
                        _search.clear();
                        setState(() {});
                      },
                    ),
                ],
                onChanged: (_) => setState(() {}),
              ),
            ),
            TabBar(
              controller: _tabs,
              tabs: const [
                Tab(text: 'Featured'),
                Tab(text: 'Games'),
                Tab(text: 'Apps'),
                Tab(text: 'Categories'),
              ],
            ),
            Expanded(child: body),
          ],
        ),
      ),
    );
  }
}

// ───────────────────────── Grid + Cards ─────────────────────────

class AppGrid extends StatelessWidget {
  const AppGrid({
    super.key,
    required this.apps,
    required this.onRefresh,
    this.banner = const [],
  });

  final List<ModApp> apps, banner;
  final Future<void> Function() onRefresh;

  @override
  Widget build(BuildContext context) {
    return RefreshIndicator(
      onRefresh: onRefresh,
      child: CustomScrollView(
        physics: const AlwaysScrollableScrollPhysics(),
        slivers: [
          if (banner.isNotEmpty)
            SliverToBoxAdapter(child: FeaturedBanner(items: banner)),
          if (apps.isEmpty)
            const SliverFillRemaining(
              hasScrollBody: false,
              child: Center(child: Text('Kuch nahi mila')),
            )
          else
            SliverPadding(
              padding: const EdgeInsets.all(12),
              sliver: SliverGrid.builder(
                gridDelegate: const SliverGridDelegateWithMaxCrossAxisExtent(
                  maxCrossAxisExtent: 200,
                  mainAxisExtent: 224,
                  crossAxisSpacing: 12,
                  mainAxisSpacing: 12,
                ),
                itemCount: apps.length,
                itemBuilder: (_, i) => AppCard(app: apps[i]),
              ),
            ),
        ],
      ),
    );
  }
}

class AppCard extends StatelessWidget {
  const AppCard({super.key, required this.app});
  final ModApp app;

  @override
  Widget build(BuildContext context) {
    final cs = Theme.of(context).colorScheme;
    return Card(
      elevation: 0,
      color: cs.surfaceContainerHighest.withAlpha(90),
      clipBehavior: Clip.antiAlias,
      shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(16)),
      child: InkWell(
        onTap: () => openUrl(context, app.link, inApp: true),
        child: Padding(
          padding: const EdgeInsets.all(12),
          child: Column(
            children: [
              ClipRRect(
                borderRadius: BorderRadius.circular(18),
                child: AppIcon(url: app.icon, size: 84),
              ),
              Expanded(
                child: Center(
                  child: Text(
                    app.title,
                    maxLines: 2,
                    overflow: TextOverflow.ellipsis,
                    textAlign: TextAlign.center,
                    style: const TextStyle(
                        fontWeight: FontWeight.w600, fontSize: 13.5),
                  ),
                ),
              ),
              SizedBox(
                width: double.infinity,
                child: FilledButton(
                  onPressed: () => openUrl(context, app.download),
                  style: FilledButton.styleFrom(
                    minimumSize: const Size.fromHeight(36),
                    padding: EdgeInsets.zero,
                  ),
                  child: const Text('Download'),
                ),
              ),
            ],
          ),
        ),
      ),
    );
  }
}

class AppIcon extends StatelessWidget {
  const AppIcon({super.key, required this.url, required this.size});
  final String url;
  final double size;

  @override
  Widget build(BuildContext context) {
    final fallback = Container(
      width: size,
      height: size,
      color: Theme.of(context).colorScheme.surfaceContainerHighest,
      child: Icon(Icons.android, size: size / 2, color: kBrand),
    );
    if (url.isEmpty) return fallback;
    return CachedNetworkImage(
      imageUrl: url,
      width: size,
      height: size,
      fit: BoxFit.cover,
      placeholder: (_, __) => fallback,
      errorWidget: (_, __, ___) => fallback,
    );
  }
}

// ───────────────────────── Featured banner ─────────────────────────

class FeaturedBanner extends StatefulWidget {
  const FeaturedBanner({super.key, required this.items});
  final List<ModApp> items;

  @override
  State<FeaturedBanner> createState() => _FeaturedBannerState();
}

class _FeaturedBannerState extends State<FeaturedBanner> {
  final PageController _ctrl = PageController(viewportFraction: 0.92);
  Timer? _timer;
  int _page = 0;

  @override
  void initState() {
    super.initState();
    _timer = Timer.periodic(const Duration(seconds: 4), (_) {
      if (!_ctrl.hasClients || widget.items.length < 2) return;
      final next = (_page + 1) % widget.items.length;
      _ctrl.animateToPage(
        next,
        duration: const Duration(milliseconds: 400),
        curve: Curves.easeInOut,
      );
    });
  }

  @override
  void dispose() {
    _timer?.cancel();
    _ctrl.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        const SizedBox(height: 12),
        SizedBox(
          height: 176,
          child: PageView.builder(
            controller: _ctrl,
            itemCount: widget.items.length,
            onPageChanged: (i) => setState(() => _page = i),
            itemBuilder: (_, i) {
              final app = widget.items[i];
              return GestureDetector(
                onTap: () => openUrl(context, app.link, inApp: true),
                child: Padding(
                  padding: const EdgeInsets.symmetric(horizontal: 6),
                  child: ClipRRect(
                    borderRadius: BorderRadius.circular(20),
                    child: Stack(
                      fit: StackFit.expand,
                      children: [
                        app.cover.isEmpty
                            ? Container(color: kBrand)
                            : CachedNetworkImage(
                                imageUrl: app.cover,
                                fit: BoxFit.cover,
                                errorWidget: (_, __, ___) =>
                                    Container(color: kBrand),
                              ),
                        const DecoratedBox(
                          decoration: BoxDecoration(
                            gradient: LinearGradient(
                              begin: Alignment.topCenter,
                              end: Alignment.bottomCenter,
                              colors: [Colors.transparent, Colors.black87],
                            ),
                          ),
                        ),
                        Positioned(
                          left: 16,
                          right: 16,
                          bottom: 14,
                          child: Text(
                            app.title,
                            maxLines: 2,
                            overflow: TextOverflow.ellipsis,
                            style: const TextStyle(
                              color: Colors.white,
                              fontSize: 18,
                              fontWeight: FontWeight.w700,
                            ),
                          ),
                        ),
                      ],
                    ),
                  ),
                ),
              );
            },
          ),
        ),
        const SizedBox(height: 8),
        Row(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            for (var i = 0; i < widget.items.length; i++)
              AnimatedContainer(
                duration: const Duration(milliseconds: 250),
                margin: const EdgeInsets.symmetric(horizontal: 3),
                width: i == _page ? 18 : 6,
                height: 6,
                decoration: BoxDecoration(
                  color: i == _page ? kBrand : Colors.grey.shade400,
                  borderRadius: BorderRadius.circular(3),
                ),
              ),
          ],
        ),
      ],
    );
  }
}

// ───────────────────────── Categories ─────────────────────────

class CategoriesTab extends StatelessWidget {
  const CategoriesTab({
    super.key,
    required this.apps,
    required this.query,
    required this.onRefresh,
  });

  final List<ModApp> apps;
  final String query;
  final Future<void> Function() onRefresh;

  @override
  Widget build(BuildContext context) {
    final map = <String, List<ModApp>>{};
    for (final a in apps) {
      for (final l in a.labels) {
        map.putIfAbsent(l, () => []).add(a);
      }
    }
    final names = map.keys
        .where((n) => query.isEmpty || n.toLowerCase().contains(query))
        .toList()
      ..sort((a, b) => a.toLowerCase().compareTo(b.toLowerCase()));

    return RefreshIndicator(
      onRefresh: onRefresh,
      child: names.isEmpty
          ? ListView(
              physics: const AlwaysScrollableScrollPhysics(),
              children: const [
                SizedBox(height: 120),
                Center(child: Text('Koi category nahi mili')),
              ],
            )
          : GridView.builder(
              physics: const AlwaysScrollableScrollPhysics(),
              padding: const EdgeInsets.all(12),
              gridDelegate: const SliverGridDelegateWithMaxCrossAxisExtent(
                maxCrossAxisExtent: 220,
                mainAxisExtent: 84,
                crossAxisSpacing: 12,
                mainAxisSpacing: 12,
              ),
              itemCount: names.length,
              itemBuilder: (context, i) {
                final name = names[i];
                final list = map[name]!;
                final cs = Theme.of(context).colorScheme;
                return Material(
                  color: cs.surfaceContainerHighest.withAlpha(90),
                  borderRadius: BorderRadius.circular(16),
                  child: InkWell(
                    borderRadius: BorderRadius.circular(16),
                    onTap: () => Navigator.of(context).push(
                      MaterialPageRoute(
                        builder: (_) =>
                            CategoryPage(title: name, apps: list),
                      ),
                    ),
                    child: Padding(
                      padding: const EdgeInsets.symmetric(horizontal: 14),
                      child: Row(
                        children: [
                          const Icon(Icons.folder_outlined, color: kBrand),
                          const SizedBox(width: 10),
                          Expanded(
                            child: Column(
                              mainAxisAlignment: MainAxisAlignment.center,
                              crossAxisAlignment: CrossAxisAlignment.start,
                              children: [
                                Text(name,
                                    maxLines: 1,
                                    overflow: TextOverflow.ellipsis,
                                    style: const TextStyle(
                                        fontWeight: FontWeight.w600)),
                                Text('${list.length} items',
                                    style: Theme.of(context)
                                        .textTheme
                                        .bodySmall),
                              ],
                            ),
                          ),
                        ],
                      ),
                    ),
                  ),
                );
              },
            ),
    );
  }
}

class CategoryPage extends StatelessWidget {
  const CategoryPage({super.key, required this.title, required this.apps});
  final String title;
  final List<ModApp> apps;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text(title)),
      body: AppGrid(apps: apps, onRefresh: () async {}),
    );
  }
}
