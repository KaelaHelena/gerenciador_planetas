import 'package:flutter/material.dart';
// ignore: depend_on_referenced_packages
import 'package:path/path.dart';
import 'package:sqflite/sqflite.dart';
import 'package:provider/provider.dart';

/// Modelo (Model)
class Planeta {
  final int id;
  final String nome;
  final double distanciaDoSol;
  final double tamanho;
  final String? apelido;

  Planeta({
    required this.id,
    required this.nome,
    required this.distanciaDoSol,
    required this.tamanho,
    this.apelido,
  });

  Map<String, dynamic> toMap() {
    return {
      'id': id,
      'nome': nome,
      'distanciaDoSol': distanciaDoSol,
      'tamanho': tamanho,
      'apelido': apelido,
    };
  }

  static Planeta fromMap(Map<String, dynamic> map) {
    return Planeta(
      id: map['id'],
      nome: map['nome'],
      distanciaDoSol: map['distanciaDoSol'],
      tamanho: map['tamanho'],
      apelido: map['apelido'],
    );
  }
}

/// Banco de Dados (Database Helper)
class DatabaseHelper {
  static final DatabaseHelper instance = DatabaseHelper._init();
  static Database? _database;

  DatabaseHelper._init();

  Future<Database> get database async {
    if (_database != null) return _database!;
    _database = await _initDB('planetas.db');
    return _database!;
  }

  Future<Database> _initDB(String filePath) async {
    final dbPath = await getDatabasesPath();
    final path = join(dbPath, filePath);
    return await openDatabase(path, version: 1, onCreate: _createDB);
  }

  Future _createDB(Database db, int version) async {
    await db.execute(''' 
    CREATE TABLE planetas (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      nome TEXT NOT NULL,
      distanciaDoSol REAL NOT NULL,
      tamanho REAL NOT NULL,
      apelido TEXT
    )
    ''');

  }

  Future<void> inserirPlaneta(Planeta planeta) async {
    final db = await instance.database;
    await db.insert('planetas', planeta.toMap());
  }

  Future<void> atualizarPlaneta(Planeta planeta) async {
    final db = await instance.database;
    await db.update(
      'planetas',
      planeta.toMap(),
      where: 'id = ?',
      whereArgs: [planeta.id],
    );
  }

  Future<void> removerPlaneta(int id) async {
    final db = await instance.database;
    await db.delete(
      'planetas',
      where: 'id = ?',
      whereArgs: [id],
    );
  }

  Future<List<Planeta>> listarPlanetas() async {
    final db = await instance.database;
    final result = await db.query('planetas');
    return result.map((json) => Planeta.fromMap(json)).toList();
  }
}

/// ViewModel
class PlanetaViewModel extends ChangeNotifier {
  List<Planeta> _planetas = [];

  List<Planeta> get planetas => _planetas;

  Future<void> carregarPlanetas() async {
    _planetas = await DatabaseHelper.instance.listarPlanetas();
    notifyListeners();
  }

  Future<void> adicionarPlaneta(Planeta planeta) async {
    await DatabaseHelper.instance.inserirPlaneta(planeta);
    await carregarPlanetas();
  }

  Future<void> atualizarPlaneta(Planeta planeta) async {
    await DatabaseHelper.instance.atualizarPlaneta(planeta);
    await carregarPlanetas();
  }

  Future<void> removerPlaneta(int id) async {
    await DatabaseHelper.instance.removerPlaneta(id);
    await carregarPlanetas();
  }
}

/// Interface do Usuário (View)
void main() {
  runApp(MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return ChangeNotifierProvider(
      create: (_) => PlanetaViewModel()..carregarPlanetas(),
      child: MaterialApp(
        routes: {
          '/': (context) => TelaInicial(),
          '/planetas': (context) => PlanetaListaScreen(),
        },
        initialRoute: '/',
      ),
    );
  }
}

class TelaInicial extends StatelessWidget {
  const TelaInicial({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('Tela Inicial')),
      body: Center(
        child: ElevatedButton(
          style: ElevatedButton.styleFrom(
            backgroundColor: Colors.blue,
            padding: EdgeInsets.symmetric(horizontal: 50, vertical: 20),
            textStyle: TextStyle(fontSize: 18),
          ),
          onPressed: () {
            Navigator.pushNamed(context, '/planetas');
          },
          child: Text('Iniciar'),
        ),
      ),
    );
  }
}

class PlanetaListaScreen extends StatelessWidget {
  final TextEditingController _nomeController = TextEditingController();
  final TextEditingController _distanciaDoSolController = TextEditingController();
  final TextEditingController _tamanhoController = TextEditingController();
  final TextEditingController _apelidoController = TextEditingController();

  PlanetaListaScreen({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('Lista de Planetas')),
      body: Consumer<PlanetaViewModel>(builder: (context, viewModel, child) {
        return ListView.builder(
          itemCount: viewModel.planetas.length,
          itemBuilder: (context, index) {
            final planeta = viewModel.planetas[index];
            return ListTile(
              title: Text(planeta.nome),
              subtitle: Text(planeta.apelido ?? 'Sem apelido'),
              trailing: Column(
                mainAxisAlignment: MainAxisAlignment.center,
                children: [
                  Text('Distância: ${planeta.distanciaDoSol} UA'),
                  Text('Tamanho: ${planeta.tamanho} km'),
                ],
              ),
              onTap: () {
                _preencherCampos(planeta);
                _exibirDialog(context, viewModel, planeta);
              },
              // Ícone de exclusão
              leading: IconButton(
                icon: Icon(Icons.delete),
                onPressed: () {
                  _confirmarRemocao(context, viewModel, planeta.id);
                },
              ),
            );
          },
        );
      }),
      floatingActionButton: FloatingActionButton(
        onPressed: () {
          _limparCampos();
          _exibirDialog(context, context.read<PlanetaViewModel>(), null);
        },
        child: Icon(Icons.add),
      ),
    );
  }

  void _preencherCampos(Planeta planeta) {
    _nomeController.text = planeta.nome;
    _distanciaDoSolController.text = planeta.distanciaDoSol.toString();
    _tamanhoController.text = planeta.tamanho.toString();
    _apelidoController.text = planeta.apelido ?? '';
  }

  void _exibirDialog(
    BuildContext context,
    PlanetaViewModel viewModel,
    Planeta? planeta,
  ) {
    showDialog(
      context: context,
      builder: (context) => AlertDialog(
        title: Text(planeta == null ? 'Adicionar Planeta' : 'Detalhes do Planeta'),
        content: Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            TextField(controller: _nomeController, decoration: InputDecoration(labelText: 'Nome')),
            TextField(
              controller: _distanciaDoSolController,
              decoration: InputDecoration(labelText: 'Distância do Sol (UA)'),
              keyboardType: TextInputType.number,
            ),
            TextField(
              controller: _tamanhoController,
              decoration: InputDecoration(labelText: 'Tamanho (km)'),
              keyboardType: TextInputType.number,
            ),
            TextField(controller: _apelidoController, decoration: InputDecoration(labelText: 'Apelido')),
          ],
        ),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(context),
            child: Text('Cancelar'),
          ),
          TextButton(
            onPressed: () {
              final planetaAtualizado = Planeta(
                id: planeta?.id ?? 0,
                nome: _nomeController.text,
                distanciaDoSol: double.parse(_distanciaDoSolController.text),
                tamanho: double.parse(_tamanhoController.text),
                apelido: _apelidoController.text,
              );
              if (planeta == null) {
                viewModel.adicionarPlaneta(planetaAtualizado);
              } else {
                viewModel.atualizarPlaneta(planetaAtualizado);
              }
              Navigator.pop(context);
            },
            child: Text(planeta == null ? 'Adicionar' : 'Salvar'),
          ),
        ],
      ),
    );
  }

  void _confirmarRemocao(BuildContext context, PlanetaViewModel viewModel, int id) {
    showDialog(
      context: context,
      builder: (context) => AlertDialog(
        title: Text('Confirmar exclusão'),
        content: Text('Tem certeza que deseja excluir este planeta?'),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(context),
            child: Text('Cancelar'),
          ),
          TextButton(
            onPressed: () {
              viewModel.removerPlaneta(id);
              Navigator.pop(context);
            },
            child: Text('Excluir'),
          ),
        ],
      ),
    );
  }

  void _limparCampos() {
    _nomeController.clear();
    _distanciaDoSolController.clear();
    _tamanhoController.clear();
    _apelidoController.clear();
  }
}
