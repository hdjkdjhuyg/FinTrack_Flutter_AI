dependencies:
  flutter:
    sdk: flutter
  cupertino_icons: ^1.0.6
  sqflite: ^2.3.0 # لإدارة قاعدة البيانات المحلية
  path_provider: ^2.0.11 # للوصول إلى مسارات الملفات
  shared_preferences: ^2.2.0 # لحفظ التفضيلات البسيطة
  local_auth: ^2.1.7 # للمصادقة البيومترية
  encrypt: ^5.0.1 # للتشفير
  syncfusion_flutter_charts: ^23.1.47 # للرسوم البيانية (إذا كنت تخطط لاستخدامها في شاشات التقارير)
  flutter_secure_storage: ^8.0.0 # لتخزين مفاتيح التشفير بأمان
  flutter_local_notifications: ^15.1.1 # للإشعارات المحلية
  intl: ^0.18.0 # لتنسيق التاريخ والعملات
  connectivity_plus: ^4.0.2 # للتحقق من الاتصال بالإنترنت (إذا كان الكود يعتمد عليه لاحقاً)
  share_plus: ^7.0.2 # للمشاركة
  pdf: ^3.10.4 # لإنشاء ملفات PDF
  printing: ^5.11.1 # لطباعة ومشاركة ملفات PDF
Import 'package:flutter/material.dart';
import 'package:sqflite/sqflite.dart';
import 'package:path_provider/path_provider.dart';
import 'package:shared_preferences/shared_preferences.dart';
import 'package:local_auth/local_auth.dart';
import 'package:encrypt/encrypt.dart' as encrypt;
import 'package:syncfusion_flutter_charts/charts.dart';
import 'package:flutter_secure_storage/flutter_secure_storage.dart';
import 'package:flutter_local_notifications/flutter_local_notifications.dart';
import 'package:intl/intl.dart';
import 'package:connectivity_plus/connectivity_plus.dart';
import 'package:share_plus/share_plus.dart';
import 'package:pdf/pdf.dart';
import 'package:pdf/widgets.dart' as pw;
import 'package:printing/printing.dart';
import 'dart:io';
import 'dart:convert';

// -----------------------------------------------------------------------------
// 🔹 1️⃣ Main Function - نقطة بداية التطبيق
// -----------------------------------------------------------------------------
void main() async {
  WidgetsFlutterBinding.ensureInitialized();

  // تهيئة إعدادات الإشعارات المحلية
  final FlutterLocalNotificationsPlugin notificationsPlugin = FlutterLocalNotificationsPlugin();
  const AndroidInitializationSettings androidSettings =
      AndroidInitializationSettings('app_icon'); // تأكد من وجود app_icon في res/drawable
  const InitializationSettings settings = InitializationSettings(android: androidSettings);
  await notificationsPlugin.initialize(settings);

  // طلب إذن الإشعارات لأندرويد 13 فما فوق
  if (Platform.isAndroid) {
    await notificationsPlugin
        .resolvePlatformSpecificImplementation<
            AndroidFlutterLocalNotificationsPlugin>()
        ?.requestNotificationsPermission();
  }

  runApp(const FinancialManagerApp());
}

// -----------------------------------------------------------------------------
// 🔹 2️⃣ FinancialManagerApp - التطبيق الرئيسي
// -----------------------------------------------------------------------------
class FinancialManagerApp extends StatelessWidget {
  const FinancialManagerApp({super.key});

  @override
  Widget build(BuildContext context) {
    return FutureBuilder<bool>(
      future: _getThemePreference(),
      builder: (context, snapshot) {
        final isDarkMode = snapshot.data ?? false;
        return MaterialApp(
          title: 'إدارة المالية الشخصية',
          theme: ThemeData(
            primarySwatch: Colors.green,
            brightness: Brightness.light,
            useMaterial3: true,
          ),
          darkTheme: ThemeData(
            primarySwatch: Colors.green,
            brightness: Brightness.dark,
            useMaterial3: true,
          ),
          themeMode: isDarkMode ? ThemeMode.dark : ThemeMode.light,
          home: const SecurityScreen(),
        );
      },
    );
  }

  static Future<bool> _getThemePreference() async {
    final prefs = await SharedPreferences.getInstance();
    return prefs.getBool('isDarkMode') ?? false;
  }
}

// -----------------------------------------------------------------------------
// 🔹 3️⃣ Models - نماذج البيانات
// -----------------------------------------------------------------------------
class SpendingHabit {
  final int id;
  final String amount; // مشفر
  final String category;
  final int frequency;
  final String date;

  SpendingHabit({
    required this.id,
    required this.amount,
    required this.category,
    required this.frequency,
    required this.date,
  });

  factory SpendingHabit.fromMap(
      Map<String, dynamic> map, encrypt.Encrypter encrypter, encrypt.IV iv) {
    return SpendingHabit(
      id: map['id'],
      amount: encrypter.decrypt64(map['amount'], iv: iv),
      category: map['category'],
      frequency: map['frequency'],
      date: map['date'],
    );
  }

  Map<String, dynamic> toMap(encrypt.Encrypter encrypter, encrypt.IV iv) {
    return {
      'id': id,
      'amount': encrypter.encrypt(amount, iv: iv).base64,
      'category': category,
      'frequency': frequency,
      'date': date,
    };
  }
}

class Loan {
  final int id;
  final String friend;
  final String amount; // مشفر
  final bool isPaid;

  Loan({
    required this.id,
    required this.friend,
    required this.amount,
    required this.isPaid,
  });

  factory Loan.fromMap(
      Map<String, dynamic> map, encrypt.Encrypter encrypter, encrypt.IV iv) {
    return Loan(
      id: map['id'],
      friend: map['friend'],
      amount: encrypter.decrypt64(map['amount'], iv: iv),
      isPaid: map['isPaid'] == 1,
    );
  }

  Map<String, dynamic> toMap(encrypt.Encrypter encrypter, encrypt.IV iv) {
    return {
      'id': id,
      'friend': friend,
      'amount': encrypter.encrypt(amount, iv: iv).base64,
      'isPaid': isPaid ? 1 : 0,
    };
  }
}

class Budget {
  final int id;
  final String category;
  final double maxAmount;
  final String period; // 'monthly' or 'weekly'

  Budget({
    required this.id,
    required this.category,
    required this.maxAmount,
    required this.period,
  });

  factory Budget.fromMap(Map<String, dynamic> map) {
    return Budget(
      id: map['id'],
      category: map['category'],
      maxAmount: map['maxAmount'],
      period: map['period'],
    );
  }

  Map<String, dynamic> toMap() {
    return {
      'id': id,
      'category': category,
      'maxAmount': maxAmount,
      'period': period,
    };
  }
}

class Income {
  final int id;
  final String source;
  final String amount; // مشفر
  final String date;

  Income({
    required this.id,
    required this.source,
    required this.amount,
    required this.date,
  });

  factory Income.fromMap(
      Map<String, dynamic> map, encrypt.Encrypter encrypter, encrypt.IV iv) {
    return Income(
      id: map['id'],
      source: map['source'],
      amount: encrypter.decrypt64(map['amount'], iv: iv),
      date: map['date'],
    );
  }

  Map<String, dynamic> toMap(encrypt.Encrypter encrypter, encrypt.IV iv) {
    return {
      'id': id,
      'source': source,
      'amount': encrypter.encrypt(amount, iv: iv).base64,
      'date': date,
    };
  }
}

class SavingsGoal {
  final int id;
  final String goalName;
  final String targetAmount; // مشفر
  final String currentAmount; // مشفر

  SavingsGoal({
    required this.id,
    required this.goalName,
    required this.targetAmount,
    required this.currentAmount,
  });

  factory SavingsGoal.fromMap(
      Map<String, dynamic> map, encrypt.Encrypter encrypter, encrypt.IV iv) {
    return SavingsGoal(
      id: map['id'],
      goalName: map['goalName'],
      targetAmount: encrypter.decrypt64(map['targetAmount'], iv: iv),
      currentAmount: encrypter.decrypt64(map['currentAmount'], iv: iv),
    );
  }

  Map<String, dynamic> toMap(encrypt.Encrypter encrypter, encrypt.IV iv) {
    return {
      'id': id,
      'goalName': goalName,
      'targetAmount': encrypter.encrypt(targetAmount, iv: iv).base64,
      'currentAmount': encrypter.encrypt(currentAmount, iv: iv).base64,
    };
  }
}

class Bill {
  final int id;
  final String name;
  final String amount; // مشفر
  final String dueDate;
  final bool isPaid;

  Bill({
    required this.id,
    required this.name,
    required this.amount,
    required this.dueDate,
    required this.isPaid,
  });

  factory Bill.fromMap(
      Map<String, dynamic> map, encrypt.Encrypter encrypter, encrypt.IV iv) {
    return Bill(
      id: map['id'],
      name: map['name'],
      amount: encrypter.decrypt64(map['amount'], iv: iv),
      dueDate: map['dueDate'],
      isPaid: map['isPaid'] == 1,
    );
  }

  Map<String, dynamic> toMap(encrypt.Encrypter encrypter, encrypt.IV iv) {
    return {
      'id': id,
      'name': name,
      'amount': encrypter.encrypt(amount, iv: iv).base64,
      'dueDate': dueDate,
      'isPaid': isPaid ? 1 : 0,
    };
  }
}

class Category {
  final int id;
  final String name;
  final String icon; // يمكن أن يكون اسم أيقونة أو مسار صورة

  Category({
    required this.id,
    required this.name,
    required this.icon,
  });

  factory Category.fromMap(Map<String, dynamic> map) {
    return Category(
      id: map['id'],
      name: map['name'],
      icon: map['icon'],
    );
  }

  Map<String, dynamic> toMap() {
    return {
      'id': id,
      'name': name,
      'icon': icon,
    };
  }
}

// -----------------------------------------------------------------------------
// 🔹 4️⃣ SecurityScreen - شاشة المصادقة البيومترية
// -----------------------------------------------------------------------------
class SecurityScreen extends StatefulWidget {
  const SecurityScreen({super.key});

  @override
  _SecurityScreenState createState() => _SecurityScreenState();
}

class _SecurityScreenState extends State<SecurityScreen> {
  final LocalAuthentication auth = LocalAuthentication();
  bool _isLoading = false;

  @override
  void initState() {
    super.initState();
    // يمكنك هنا استدعاء _authenticate() تلقائياً إذا أردت فرض المصادقة عند البدء
    // _authenticate();
  }

  Future<void> _authenticate() async {
    setState(() => _isLoading = true);
    try {
      bool canCheckBiometrics = await auth.canCheckBiometrics;
      if (!canCheckBiometrics) {
        ScaffoldMessenger.of(context).showSnackBar(
          const SnackBar(content: Text("البصمة غير مدعومة على هذا الجهاز")),
        );
        Navigator.pushReplacement(
          context,
          MaterialPageRoute(builder: (context) => const HomeScreen()),
        );
        return;
      }

      bool authenticated = await auth.authenticate(
        localizedReason: 'يرجى تأكيد هويتك للدخول',
        options: const AuthenticationOptions(
          biometricOnly: true,
          stickyAuth: true,
        ),
      );

      if (authenticated) {
        Navigator.pushReplacement(
          context,
          MaterialPageRoute(builder: (context) => const HomeScreen()),
        );
      } else {
        ScaffoldMessenger.of(context).showSnackBar(
          const SnackBar(content: Text("فشل المصادقة، حاول مرة أخرى")),
        );
      }
    } catch (e) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text("خطأ أثناء المصادقة: $e")),
      );
    } finally {
      setState(() => _isLoading = false);
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text("تسجيل الدخول الآمن")),
      body: Center(
        child: _isLoading
            ? const CircularProgressIndicator()
            : Column(
                mainAxisAlignment: MainAxisAlignment.center,
                children: [
                  ElevatedButton(
                    onPressed: _authenticate,
                    child: const Text("تسجيل الدخول بالبصمة"),
                  ),
                  const SizedBox(height: 16),
                  TextButton(
                    onPressed: () {
                      Navigator.pushReplacement(
                        context,
                        MaterialPageRoute(builder: (context) => const HomeScreen()),
                      );
                    },
                    child: const Text("تخطي المصادقة"),
                  ),
                ],
              ),
      ),
    );
  }
}

// -----------------------------------------------------------------------------
// 🔹 5️⃣ HomeScreen - الشاشة الرئيسية لإدارة المالية
// -----------------------------------------------------------------------------
class HomeScreen extends StatefulWidget {
  const HomeScreen({super.key});

  @override
  _HomeScreenState createState() => _HomeScreenState();
}

class _HomeScreenState extends State<HomeScreen> {
  late Database _database;
  late encrypt.Encrypter encrypter;
  late encrypt.Key key;
  late encrypt.IV iv;
  List<SpendingHabit> _spendingData = [];
  List<Loan> _loansData = [];
  List<Budget> _budgetsData = [];
  List<Income> _incomesData = [];
  List<SavingsGoal> _savingsGoalsData = [];
  List<Bill> _billsData = [];
  List<Category> _categoriesData = [];
  bool _isDarkMode = false;
  bool _isStealthMode = false;
  bool _isLoading = false;
  String _language = 'ar'; // الافتراضي هو العربية
  final FlutterLocalNotificationsPlugin notificationsPlugin = FlutterLocalNotificationsPlugin();
  final _secureStorage = const FlutterSecureStorage();

  static const String _backupKey = 'backup_data'; // مفتاح للنسخ الاحتياطي المحلي

  @override
  void initState() {
    super.initState();
    _loadPreferences();
    _initializeEncryptionAndDatabase();
  }

  @override
  void dispose() {
    // حاول إغلاق قاعدة البيانات فقط إذا كانت مفتوحة
    // تجنب الوصول إلى _database إذا كانت لا تزال في حالة تهيئة
    try {
      if (_database.isOpen) {
        _database.close();
      }
    } catch (e) {
      debugPrint("Error closing database: $e");
    }
    super.dispose();
  }

  Future<void> _loadPreferences() async {
    final prefs = await SharedPreferences.getInstance();
    setState(() {
      _isDarkMode = prefs.getBool('isDarkMode') ?? false;
      _isStealthMode = prefs.getBool('isStealthMode') ?? false;
      _language = prefs.getString('language') ?? 'ar';
    });
  }

  Future<void> _setThemePreference(bool value) async {
    final prefs = await SharedPreferences.getInstance();
    await prefs.setBool('isDarkMode', value);
    setState(() {
      _isDarkMode = value;
      // إعادة تشغيل التطبيق لتطبيق الثيم الجديد فوراً
      runApp(const FinancialManagerApp());
    });
  }

  Future<void> _setStealthMode(bool value) async {
    final prefs = await SharedPreferences.getInstance();
    await prefs.setBool('isStealthMode', value);
    setState(() {
      _isStealthMode = value;
    });
  }

  Future<void> _setLanguage(String value) async {
    final prefs = await SharedPreferences.getInstance();
    await prefs.setString('language', value);
    setState(() {
      _language = value;
    });
    // قد تحتاج إلى إعادة تشغيل التطبيق أو تحديث الواجهة بشكل أعمق
    // لكي يتم تطبيق تغيير اللغة على جميع النصوص فوراً.
  }

  void _showLanguageDialog() {
    showDialog(
      context: context,
      builder: (context) {
        return AlertDialog(
          title: Text(_language == 'ar' ? 'اختر اللغة' : 'Select Language'),
          content: Column(
            mainAxisSize: MainAxisSize.min,
            children: [
              RadioListTile<String>(
                title: const Text('العربية'),
                value: 'ar',
                groupValue: _language,
                onChanged: (value) {
                  if (value != null) {
                    _setLanguage(value);
                    Navigator.pop(context);
                  }
                },
              ),
              RadioListTile<String>(
                title: const Text('English'),
                value: 'en',
                groupValue: _language,
                onChanged: (value) {
                  if (value != null) {
                    _setLanguage(value);
                    Navigator.pop(context);
                  }
                },
              ),
            ],
          ),
        );
      },
    );
  }

  Future<void> _initializeEncryptionAndDatabase() async {
    setState(() => _isLoading = true);
    try {
      // تهيئة التشفير
      String? storedKey = await _secureStorage.read(key: 'encryption_key');
      String? storedIV = await _secureStorage.read(key: 'encryption_iv');
      if (storedKey == null || storedIV == null) {
        key = encrypt.Key.fromSecureRandom(32);
        iv = encrypt.IV.fromSecureRandom(16);
        await _secureStorage.write(key: 'encryption_key', value: key.base64);
        await _secureStorage.write(key: 'encryption_iv', value: iv.base64);
      } else {
        key = encrypt.Key.fromBase64(storedKey);
        iv = encrypt.IV.fromBase64(storedIV);
      }
      encrypter = encrypt.Encrypter(encrypt.AES(key));

      // تهيئة قاعدة البيانات
      Directory directory = await getApplicationDocumentsDirectory();
      String path = '${directory.path}/financial_manager.db';

      _database = await openDatabase(
        path,
        version: 1, // رقم الإصدار الأولي
        onCreate: (db, version) async {
          await db.execute(
            "CREATE TABLE spending(id INTEGER PRIMARY KEY AUTOINCREMENT, amount TEXT, category TEXT, frequency INTEGER, date TEXT)",
          );
          await db.execute(
            "CREATE TABLE loans(id INTEGER PRIMARY KEY AUTOINCREMENT, friend TEXT, amount TEXT, isPaid INTEGER)",
          );
          await db.execute(
            "CREATE TABLE budgets(id INTEGER PRIMARY KEY AUTOINCREMENT, category TEXT, maxAmount REAL, period TEXT)",
          );
          await db.execute(
            "CREATE TABLE incomes(id INTEGER PRIMARY KEY AUTOINCREMENT, source TEXT, amount TEXT, date TEXT)",
          );
          await db.execute(
            "CREATE TABLE savings_goals(id INTEGER PRIMARY KEY AUTOINCREMENT, goalName TEXT, targetAmount TEXT, currentAmount TEXT)",
          );
          await db.execute(
            "CREATE TABLE bills(id INTEGER PRIMARY KEY AUTOINCREMENT, name TEXT, amount TEXT, dueDate TEXT, isPaid INTEGER)",
          );
          await db.execute(
            "CREATE TABLE categories(id INTEGER PRIMARY KEY AUTOINCREMENT, name TEXT, icon TEXT)",
          );
        },
        // إذا كان لديك إصدارات سابقة من قاعدة البيانات وتحتاج لترحيلها، يمكنك استخدام onUpgrade
        // onUpgrade: (db, oldVersion, newVersion) async {
        //   if (oldVersion < 2) {
        //     await db.execute("CREATE TABLE budgets(id INTEGER PRIMARY KEY AUTOINCREMENT, category TEXT, maxAmount REAL, period TEXT)");
        //     // ... أضف جداول أخرى هنا
        //   }
        // },
      );

      // جلب جميع البيانات بعد التهيئة
      await _fetchSpendingData();
      await _fetchLoansData();
      await _fetchBudgetsData();
      await _fetchIncomesData();
      await _fetchSavingsGoalsData();
      await _fetchBillsData();
      await _fetchCategoriesData();

      // فحص التنبيهات بعد جلب البيانات
      await _checkBudgetAlerts();
      await _checkBillReminders();
    } catch (e) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text(_language == 'ar' ? 'خطأ أثناء تهيئة التطبيق: $e' : 'Error initializing app: $e')),
      );
    } finally {
      setState(() => _isLoading = false);
    }
  }

  String _formatDate(String isoDate) {
    try {
      return DateFormat('dd/MM/yyyy').format(DateTime.parse(isoDate));
    } catch (e) {
      return isoDate; // في حالة وجود خطأ في تنسيق التاريخ
    }
  }

  // -----------------------------------------------------------------------------
  // 🔹 6️⃣ Data Management - إضافة/حذف/جلب البيانات
  // -----------------------------------------------------------------------------

  Future<void> _fetchSpendingData() async {
    try {
      final List<Map<String, dynamic>> data =
          await _database.query("spending", orderBy: "date DESC");
      setState(() {
        _spendingData =
            data.map((item) => SpendingHabit.fromMap(item, encrypter, iv)).toList();
      });
    } catch (e) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text(_language == 'ar' ? 'خطأ أثناء جلب بيانات الإنفاق: $e' : 'Error fetching spending data: $e')),
      );
    }
  }

  Future<void> _addSpending(String amount, String category, int frequency, String date) async {
    try {
      await _database.insert(
        'spending',
        {'amount': amount, 'category': category, 'frequency': frequency, 'date': date},
      );
      await _fetchSpendingData();
      await _checkBudgetAlerts();
    } catch (e) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text(_language == 'ar' ? 'خطأ أثناء إضافة الإنفاق: $e' : 'Error adding spending: $e')),
      );
    }
  }

  Future<void> _deleteSpending(int id) async {
    try {
      await _database.delete('spending', where: 'id = ?', whereArgs: [id]);
      await _fetchSpendingData();
    } catch (e) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text(_language == 'ar' ? 'خطأ أثناء حذف الإنفاق: $e' : 'Error deleting spending: $e')),
      );
    }
  }

  Future<void> _fetchLoansData() async {
    try {
      final List<Map<String, dynamic>> data = await _database.query("loans");
      setState(() {
        _loansData = data.map((item) => Loan.fromMap(item, encrypter, iv)).toList();
      });
    } catch (e) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text(_language == 'ar' ? 'خطأ أثناء جلب بيانات القروض: $e' : 'Error fetching loans data: $e')),
      );
    }
  }

  Future<void> _addLoan(String friend, String amount, bool isPaid) async {
    try {
      await _database.insert(
        'loans',
        {'friend': friend, 'amount': amount, 'isPaid': isPaid ? 1 : 0},
      );
      await _fetchLoansData();
    } catch (e) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text(_language == 'ar' ? 'خطأ أثناء إضافة القرض: $e' : 'Error adding loan: $e')),
      );
    }
  }

  Future<void> _deleteLoan(int id) async {
    try {
      await _database.delete('loans', where: 'id = ?', whereArgs: [id]);
      await _fetchLoansData();
    } catch (e) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text(_language == 'ar' ? 'خطأ أثناء حذف القرض: $e' : 'Error deleting loan: $e')),
      );
    }
  }

  Future<void> _updateLoanStatus(int id, bool isPaid) async {
    try {
      await _database.update(
        'loans',
        {'isPaid': isPaid ? 1 : 0},
        where: 'id = ?',
        whereArgs: [id],
      );
      await _fetchLoansData();
      if (!isPaid) {
        // يمكنك إرسال تذكير هنا إذا لم يتم الدفع
        final loan = _loansData.firstWhere((l) => l.id == id);
        await _sendDebtReminder(loan.friend);
      }
    } catch (e) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text(_language == 'ar' ? 'خطأ أثناء تحديث حالة القرض: $e' : 'Error updating loan status: $e')),
      );
    }
  }

  Future<void> _fetchBudgetsData() async {
    try {
      final List<Map<String, dynamic>> data = await _database.query("budgets");
      setState(() {
        _budgetsData = data.map((item) => Budget.fromMap(item)).toList();
      });
    } catch (e) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text(_language == 'ar' ? 'خطأ أثناء جلب بيانات الميزانية: $e' : 'Error fetching budgets data: $e')),
      );
    }
  }

  Future<void> _addBudget(String category, double maxAmount, String period) async {
    try {
      await _database.insert(
        'budgets',
        {
          'category': category,
          'maxAmount': maxAmount,
          'period': period,
        },
      );
      await _fetchBudgetsData();
      await _checkBudgetAlerts();
    } catch (e) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text(_language == 'ar' ? 'خطأ أثناء إضافة الميزانية: $e' : 'Error adding budget: $e')),
      );
    }
  }

  Future<void> _deleteBudget(int id) async {
    try {
      await _database.delete('budgets', where: 'id = ?', whereArgs: [id]);
      await _fetchBudgetsData();
    } catch (e) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text(_language == 'ar' ? 'خطأ أثناء حذف الميزانية: $e' : 'Error deleting budget: $e')),
      );
    }
  }

  Future<void> _fetchIncomesData() async {
    try {
      final List<Map<String, dynamic>> data = await _database.query("incomes");
      setState(() {
        _incomesData = data.map((item) => Income.fromMap(item, encrypter, iv)).toList();
      });
    } catch (e) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text(_language == 'ar' ? 'خطأ أثناء جلب بيانات الدخل: $e' : 'Error fetching incomes data: $e')),
      );
    }
  }

  Future<void> _addIncome(String source, double amount, String date) async {
    try {
      final encryptedAmount = encrypter.encrypt(amount.toString(), iv: iv).base64;
      await _database.insert(
        'incomes',
        {
          'source': source,
          'amount': encryptedAmount,
          'date': date,
        },
      );
      await _fetchIncomesData();
    } catch (e) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text(_language == 'ar' ? 'خطأ أثناء إضافة الدخل: $e' : 'Error adding income: $e')),
      );
    }
  }

  Future<void> _deleteIncome(int id) async {
    try {
      await _database.delete('incomes', where: 'id = ?', whereArgs: [id]);
      await _fetchIncomesData();
    } catch (e) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text(_language == 'ar' ? 'خطأ أثناء حذف الدخل: $e' : 'Error deleting income: $e')),
      );
    }
  }

  Future<void> _fetchSavingsGoalsData() async {
    try {
      final List<Map<String, dynamic>> data = await _database.query("savings_goals");
      setState(() {
        _savingsGoalsData = data.map((item) => SavingsGoal.fromMap(item, encrypter, iv)).toList();
      });
    } catch (e) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text(_language == 'ar' ? 'خطأ أثناء جلب بيانات أهداف الادخار: $e' : 'Error fetching savings goals data: $e')),
      );
    }
  }

  Future<void> _addSavingsGoal(
      String goalName, double targetAmount, double currentAmount) async {
    try {
      final encryptedTarget = encrypter.encrypt(targetAmount.toString(), iv: iv).base64;
      final encryptedCurrent = encrypter.encrypt(currentAmount.toString(), iv: iv).base64;
      await _database.insert(
        'savings_goals',
        {
          'goalName': goalName,
          'targetAmount': encryptedTarget,
          'currentAmount': encryptedCurrent,
        },
      );
      await _fetchSavingsGoalsData();
    } catch (e) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text(_language == 'ar' ? 'خطأ أثناء إضافة هدف الادخار: $e' : 'Error adding savings goal: $e')),
      );
    }
  }

  Future<void> _deleteSavingsGoal(int id) async {
    try {
      await _database.delete('savings_goals', where: 'id = ?', whereArgs: [id]);
      await _fetchSavingsGoalsData();
    } catch (e) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text(_language == 'ar' ? 'خطأ أثناء حذف هدف الادخار: $e' : 'Error deleting savings goal: $e')),
      );
    }
  }

  Future<void> _fetchBillsData() async {
    try {
      final List<Map<String, dynamic>> data = await _database.query("bills");
      setState(() {
        _billsData = data.map((item) => Bill.fromMap(item, encrypter, iv)).toList();
      });
    } catch (e) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text(_language == 'ar' ? 'خطأ أثناء جلب بيانات الفواتير: $e' : 'Error fetching bills data: $e')),
      );
    }
  }

  Future<void> _addBill(String name, double amount, String dueDate, bool isPaid) async {
    try {
      final encryptedAmount = encrypter.encrypt(amount.toString(), iv: iv).base64;
      await _database.insert(
        'bills',
        {
          'name': name,
          'amount': encryptedAmount,
          'dueDate': dueDate,
          'isPaid': isPaid ? 1 : 0,
        },
      );
      await _fetchBillsData();
      if (!isPaid) {
        await _scheduleBillReminder(name, dueDate);
      }
    } catch (e) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text(_language == 'ar' ? 'خطأ أثناء إضافة الفاتورة: $e' : 'Error adding bill: $e')),
      );
    }
  }

  Future<void> _deleteBill(int id) async {
    try {
      await _database.delete('bills', where: 'id = ?', whereArgs: [id]);
      await _fetchBillsData();
    } catch (e) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text(_language == 'ar' ? 'خطأ أثناء حذف الفاتورة: $e' : 'Error deleting bill: $e')),
      );
    }
  }

  Future<void> _updateBillStatus(int id, bool isPaid) async {
    try {
      await _database.update(
        'bills',
        {'isPaid': isPaid ? 1 : 0},
        where: 'id = ?',
        whereArgs: [id],
      );
      await _fetchBillsData();
    } catch (e) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text(_language == 'ar' ? 'خطأ أثناء تحديث حالة الفاتورة: $e' : 'Error updating bill status: $e')),
      );
    }
  }

  Future<void> _fetchCategoriesData() async {
    try {
      final List<Map<String, dynamic>> data = await _database.query("categories");
      setState(() {
        _categoriesData = data.map((item) => Category.fromMap(item)).toList();
        if (_categoriesData.isEmpty) {
          // إذا لم تكن هناك فئات، أضف الافتراضية
          _addDefaultCategories();
        }
      });
    } catch (e) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text(_language == 'ar' ? 'خطأ أثناء جلب بيانات الفئات: $e' : 'Error fetching categories data: $e')),
      );
    }
  }

  Future<void> _addCategory(String name, String icon) async {
    try {
      await _database.insert(
        'categories',
        {'name': name, 'icon': icon},
      );
      await _fetchCategoriesData();
    } catch (e) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text(_language == 'ar' ? 'خطأ أثناء إضافة الفئة: $e' : 'Error adding category: $e')),
      );
    }
  }

  Future<void> _deleteCategory(int id) async {
    try {
      await _database.delete('categories', where: 'id = ?', whereArgs: [id]);
      await _fetchCategoriesData();
    } catch (e) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text(_language == 'ar' ? 'خطأ أثناء حذف الفئة: $e' : 'Error deleting category: $e')),
      );
    }
  }

  Future<void> _addDefaultCategories() async {
    await _addCategory('طعام', 'food');
    await _addCategory('مواصلات', 'transport');
    await _addCategory('ترفيه', 'entertainment');
    await _addCategory('تسوق', 'shopping');
    await _addCategory('فواتير', 'bills');
    await _addCategory('صحة', 'health');
    await _addCategory('تعليم', 'education');
    await _addCategory('منزل', 'home');
    await _addCategory('أخرى', 'other');
    // بعد إضافة الافتراضيات، أعد جلب البيانات لتحديث الـ UI
    await _fetchCategoriesData();
  }

  // -----------------------------------------------------------------------------
  // 🔹 7️⃣ Advanced Features - ميزات متقدمة
  // -----------------------------------------------------------------------------

  Future<double> _getSpendingForCategory(String category, String period) async {
    try {
      DateTime startDate;
      DateTime endDate = DateTime.now();

      if (period == 'monthly') {
        startDate = DateTime(endDate.year, endDate.month, 1);
      } else {
     
        
        // أسبوعي (من بداية الأسبوع الحالي)
                startDate = endDate.subtract(Duration(days: endDate.weekday - 1));
      }

      final spendingData = await _database.query('spending',
          where: 'category = ? AND date BETWEEN ? AND ?',
          whereArgs: [
            category,
            startDate.toIso8601String(),
            endDate.toIso8601String(),
          ]);
      double total = 0.0;
      for (var item in spendingData) {
        total += double.parse(encrypter.decrypt64(item['amount'] as String, iv: iv));
      }
      return total;
    } catch (e) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text(_language == 'ar' ? 'خطأ أثناء حساب الإنفاق: $e' : 'Error calculating spending: $e')),
      );
      return 0.0;
    }
  }

  Future<void> _checkBudgetAlerts() async {
    final budgets = _budgetsData;
    for (var budget in budgets) {
      final category = budget.category;
      final maxAmount = budget.maxAmount;
      final period = budget.period;
      final spent = await _getSpendingForCategory(category, period);

      if (spent >= maxAmount * 0.9) {
        const androidDetails = AndroidNotificationDetails(
          'budget_alert',
          'تنبيه الميزانية',
          channelDescription: 'تنبيهات عند اقتراب تجاوز الميزانية',
          importance: Importance.high,
        );
        const details = NotificationDetails(android: androidDetails);
        await notificationsPlugin.show(
          category.hashCode,
          _language == 'ar' ? 'تحذير الميزانية' : 'Budget Warning',
          _language == 'ar'
              ? 'إنفاقك على $category وصل إلى ${((spent / maxAmount) * 100).toStringAsFixed(1)}%'
              : 'Your spending on $category reached ${((spent / maxAmount) * 100).toStringAsFixed(1)}%',
          details,
        );
      }
    }
  }

  Future<void> _scheduleBillReminder(String name, String dueDate) async {
    try {
      final due = DateTime.parse(dueDate);
      final now = DateTime.now();
      if (due.isAfter(now)) {
        // جدولة التذكير قبل 3 أيام من تاريخ الاستحقاق
        final scheduledDate = due.subtract(const Duration(days: 3));
        if (scheduledDate.isAfter(now)) {
          const androidDetails = AndroidNotificationDetails(
            'bill_reminder_channel',
            'تذكير بالفواتير',
            channelDescription: 'تنبيهات لتذكير الفواتير المستحقة',
            importance: Importance.high,
          );
          const details = NotificationDetails(android: androidDetails);
          await notificationsPlugin.schedule(
            name.hashCode + 1, // معرف فريد
            _language == 'ar' ? 'تذكير بالفاتورة' : 'Bill Reminder',
            _language == 'ar'
                ? 'فاتورة "$name" تستحق في ${DateFormat('dd/MM/yyyy').format(due)}'
                : 'Bill "$name" is due on ${DateFormat('dd/MM/yyyy').format(due)}',
            scheduledDate,
            details,
            androidScheduleMode: AndroidScheduleMode.exactAllowWhileIdle,
          );
        }
      }
    } catch (e) {
      debugPrint("Error scheduling bill reminder: $e");
    }
  }

  Future<void> _checkBillReminders() async {
    for (var bill in _billsData) {
      if (!bill.isPaid) {
        await _scheduleBillReminder(bill.name, bill.dueDate);
      }
    }
  }

  Future<void> _sendDebtReminder(String friend) async {
    const AndroidNotificationDetails androidDetails = AndroidNotificationDetails(
      'debt_reminder_channel',
      'تذكير بالديون',
      channelDescription: 'تنبيهات لتذكيرك بالديون',
      importance: Importance.high,
    );
    const NotificationDetails details = NotificationDetails(android: androidDetails);
    await notificationsPlugin.show(
      friend.hashCode + 2, // معرف فريد
      _language == 'ar' ? 'تذكير بالدين' : 'Debt Reminder',
      _language == 'ar'
          ? 'يجب سداد الدين المستحق لـ $friend'
          : 'You need to settle the debt owed to $friend',
      details,
    );
  }

  Future<void> _transferBetweenAccounts(
      String fromAccount, String toAccount, double amount) async {
    try {
      final date = DateTime.now().toIso8601String();
      String fromCategory = '';
      String toCategory = '';

      if (fromAccount == 'incomes') {
        fromCategory = _language == 'ar' ? 'سحب من الدخل' : 'Withdraw from Income';
      } else if (fromAccount == 'savings_goals') {
        fromCategory = _language == 'ar' ? 'سحب من المدخرات' : 'Withdraw from Savings';
      } else {
        fromCategory = _language == 'ar' ? 'نقل من مصروفات عامة' : 'Transfer from General Spending';
      }

      if (toAccount == 'incomes') {
        toCategory = _language == 'ar' ? 'إضافة للدخل' : 'Add to Income';
      } else if (toAccount == 'savings_goals') {
        toCategory = _language == 'ar' ? 'إضافة للمدخرات' : 'Add to Savings';
      } else {
        toCategory = _language == 'ar' ? 'إضافة لمصروفات عامة' : 'Add to General Spending';
      }

      // تسجيل عملية سحب (مبلغ سالب)
      await _database.insert('spending', {
        'amount': encrypter.encrypt((-amount).toString(), iv: iv).base64,
        'category': fromCategory,
        'frequency': 1,
        'date': date,
      });

      // تسجيل عملية إضافة (مبلغ موجب)
      await _database.insert('spending', {
        'amount': encrypter.encrypt(amount.toString(), iv: iv).base64,
        'category': toCategory,
        'frequency': 1,
        'date': date,
      });

      await _fetchSpendingData();
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text(_language == 'ar' ? 'تم التحويل بنجاح!' : 'Transfer successful!')),
      );
    } catch (e) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text(_language == 'ar' ? 'خطأ أثناء التحويل: $e' : 'Error transferring amount: $e')),
      );
    }
  }

  Future<List<Map<String, dynamic>>> _searchTransactions(
      String query, String table,
      {DateTime? startDate, DateTime? endDate, double? minAmount, double? maxAmount}) async {
    try {
      final List<Map<String, dynamic>> rawResults = await _database.query(table);
      final List<Map<String, dynamic>> decryptedResults = rawResults.map((item) {
        // تأكد من وجود مفتاح 'amount' وأنه من نوع String قبل فك التشفير
        if (item.containsKey('amount') && item['amount'] is String) {
          return {
            ...item,
            'decryptedAmount': double.tryParse(encrypter.decrypt64(item['amount'] as String, iv: iv)) ?? 0.0,
          };
        }
        return item; // أعد العنصر كما هو إذا لم يكن يحتوي على مفتاح 'amount'
      }).toList();

      final filteredResults = decryptedResults.where((item) {
        bool matchesQuery = true;
        if (query.isNotEmpty) {
          if (table == 'spending' && item.containsKey('category')) {
            matchesQuery = (item['category'] as String).toLowerCase().contains(query.toLowerCase());
          } else if (table == 'loans' && item.containsKey('friend')) {
            matchesQuery = (item['friend'] as String).toLowerCase().contains(query.toLowerCase());
          } else if (table == 'incomes' && item.containsKey('source')) {
            matchesQuery = (item['source'] as String).toLowerCase().contains(query.toLowerCase());
          } else if (table == 'bills' && item.containsKey('name')) {
            matchesQuery = (item['name'] as String).toLowerCase().contains(query.toLowerCase());
          } else if (table == 'savings_goals' && item.containsKey('goalName')) {
            matchesQuery = (item['goalName'] as String).toLowerCase().contains(query.toLowerCase());
          }
        }

        bool matchesDate = true;
        if (item.containsKey('date') && item['date'] is String) {
          try {
            final itemDate = DateTime.parse(item['date'] as String);
            if (startDate != null && itemDate.isBefore(startDate.subtract(const Duration(days: 1)))) {
              matchesDate = false;
            }
            if (endDate != null && itemDate.isAfter(endDate.add(const Duration(days: 1)))) {
              matchesDate = false;
            }
          } catch (e) {
            matchesDate = false; // إذا كان التاريخ غير صالح، لا يتطابق
          }
        }

        bool matchesAmount = true;
        if (item.containsKey('decryptedAmount')) {
          final amount = item['decryptedAmount'] as double;
          if (minAmount != null && amount < minAmount) matchesAmount = false;
          if (maxAmount != null && amount > maxAmount) matchesAmount = false;
        }

        return matchesQuery && matchesDate && matchesAmount;
      }).toList();

      // إعادة العناصر بالشكل الأصلي مع المبلغ المشفّر إذا لم تكن هناك حاجة لـ decryptedAmount
      return filteredResults.map((item) {
        final Map<String, dynamic> cleanItem = Map.from(item);
        if (cleanItem.containsKey('decryptedAmount')) {
          cleanItem.remove('decryptedAmount'); // إزالة المفتاح الإضافي
        }
        return cleanItem;
      }).toList();

    } catch (e) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text(_language == 'ar' ? 'خطأ أثناء البحث عن المعاملات: $e' : 'Error searching transactions: $e')),
      );
      return [];
    }
  }

  Future<void> _exportReportAsPDF() async {
    try {
      final pdf = pw.Document();

      // جلب البيانات الخام من قاعدة البيانات
      final spendingData = await _database.query('spending');
      final incomesData = await _database.query('incomes');
      final loansData = await _database.query('loans');
      final budgetsData = await _database.query('budgets');
      final savingsGoalsData = await _database.query('savings_goals');
      final billsData = await _database.query('bills');

      // دالة مساعدة لإنشاء صف في الجدول
      pw.Widget _buildRow(String label, String value) {
        return pw.Row(
          children: [
            pw.Expanded(flex: 2, child: pw.Text(label, style: const pw.TextStyle(fontSize: 10))),
            pw.Expanded(flex: 3, child: pw.Text(value, style: const pw.TextStyle(fontSize: 10))),
          ],
        );
      }

      pdf.addPage(
        pw.MultiPage(
          pageFormat: PdfPageFormat.a4,
          build: (pw.Context context) {
            return [
              pw.Center(
                child: pw.Text(
                  _language == 'ar' ? 'تقرير المالية الشخصية' : 'Personal Finance Report',
                  style: pw.TextStyle(fontSize: 24, fontWeight: pw.FontWeight.bold),
                ),
              ),
              pw.SizedBox(height: 20),
              // الإنفاق
              pw.Text(_language == 'ar' ? 'الإنفاق:' : 'Spending:', style: pw.TextStyle(fontSize: 16, fontWeight: pw.FontWeight.bold)),
              pw.Divider(),
              ...spendingData.map((item) {
                return _buildRow(
                  item['category'] as String,
                  '${encrypter.decrypt64(item['amount'] as String, iv: iv)} ${_language == 'ar' ? 'جنيه' : 'EGP'} (${_formatDate(item['date'] as String)})',
                );
              }),
              pw.SizedBox(height: 10),

              // الدخل
              pw.Text(_language == 'ar' ? 'الدخل:' : 'Income:', style: pw.TextStyle(fontSize: 16, fontWeight: pw.FontWeight.bold)),
              pw.Divider(),
              ...incomesData.map((item) {
                return _buildRow(
                  item['source'] as String,
                  '${encrypter.decrypt64(item['amount'] as String, iv: iv)} ${_language == 'ar' ? 'جنيه' : 'EGP'} (${_formatDate(item['date'] as String)})',
                );
              }),
              pw.SizedBox(height: 10),

              // القروض
              pw.Text(_language == 'ar' ? 'القروض:' : 'Loans:', style: pw.TextStyle(fontSize: 16, fontWeight: pw.FontWeight.bold)),
              pw.Divider(),
              ...loansData.map((item) {
                return _buildRow(
                  item['friend'] as String,
                  '${encrypter.decrypt64(item['amount'] as String, iv: iv)} ${_language == 'ar' ? 'جنيه' : 'EGP'} (${item['isPaid'] == 1 ? (_language == 'ar' ? 'تم السداد' : 'Paid') : (_language == 'ar' ? 'لم يُسدد' : 'Unpaid')})',
                );
              }),
              pw.SizedBox(height: 10),

              // الميزانيات
              pw.Text(_language == 'ar' ? 'الميزانيات:' : 'Budgets:', style: pw.TextStyle(fontSize: 16, fontWeight: pw.FontWeight.bold)),
              pw.Divider(),
              ...budgetsData.map((item) {
                return _buildRow(
                  item['category'] as String,
                  '${item['maxAmount']} ${_language == 'ar' ? 'جنيه' : 'EGP'} (${item['period'] == 'monthly' ? (_language == 'ar' ? 'شهري' : 'Monthly') : (_language == 'ar' ? 'أسبوعي' : 'Weekly')})',
                );
              }),
              pw.SizedBox(height: 10),

              // أهداف الادخار
              pw.Text(_language == 'ar' ? 'أهداف الادخار:' : 'Savings Goals:', style: pw.TextStyle(fontSize: 16, fontWeight: pw.FontWeight.bold)),
              pw.Divider(),
              ...savingsGoalsData.map((item) {
                return _buildRow(
                  item['goalName'] as String,
                  '${encrypter.decrypt64(item['currentAmount'] as String, iv: iv)}/${encrypter.decrypt64(item['targetAmount'] as String, iv: iv)} ${_language == 'ar' ? 'جنيه' : 'EGP'}',
                );
              }),
              pw.SizedBox(height: 10),

              // الفواتير
              pw.Text(_language == 'ar' ? 'الفواتير:' : 'Bills:', style: pw.TextStyle(fontSize: 16, fontWeight: pw.FontWeight.bold)),
              pw.Divider(),
              ...billsData.map((item) {
                return _buildRow(
                  item['name'] as String,
                  '${encrypter.decrypt64(item['amount'] as String, iv: iv)} ${_language == 'ar' ? 'جنيه' : 'EGP'} (${_formatDate(item['dueDate'] as String)}) (${item['isPaid'] == 1 ? (_language == 'ar' ? 'مدفوعة' : 'Paid') : (_language == 'ar' ? 'غير مدفوعة' : 'Unpaid')})',
                );
              }),
              pw.SizedBox(height: 10),
            ];
          },
        ),
      );

      // حفظ ومشاركة ملف PDF
      final output = await pdf.save();
      await Printing.sharePdf(bytes: output, filename: 'financial_report.pdf');

      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(
          content: Text(_language == 'ar' ? 'تم تصدير التقرير بنجاح' : 'Report exported successfully'),
        ),
      );
    } catch (e) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text(_language == 'ar' ? 'خطأ أثناء تصدير التقرير: $e' : 'Error exporting report: $e')),
      );
    }
  }

  Future<void> _localBackup() async {
    try {
      final spendingData = await _database.query('spending');
      final incomesData = await _database.query('incomes');
      final loansData = await _database.query('loans');
      final budgetsData = await _database.query('budgets');
      final savingsGoalsData = await _database.query('savings_goals');
      final billsData = await _database.query('bills');
      final categoriesData = await _database.query('categories');

      final backupData = {
        'spending': spendingData,
        'incomes': incomesData,
        'loans': loansData,
        'budgets': budgetsData,
        'savings_goals': savingsGoalsData,
        'bills': billsData,
        'categories': categoriesData,
        'timestamp': DateTime.now().toIso8601String(),
      };

      final jsonString = jsonEncode(backupData);
      final prefs = await SharedPreferences.getInstance();
      await prefs.setString(_backupKey, jsonString);

      // يمكن حفظ النسخة الاحتياطية كملف أيضًا إذا أردت
      final backupDir = await getTemporaryDirectory();
      final backupFile = File('${backupDir.path}/financial_backup.json');
      await backupFile.writeAsString(jsonString);

      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(
          content: Text(_language == 'ar' ? 'تم إنشاء النسخة الاحتياطية بنجاح' : 'Backup created successfully'),
        ),
      );
    } catch (e) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text(_language == 'ar' ? 'خطأ أثناء النسخ الاحتياطي: $e' : 'Error during backup: $e')),
      );
    }
  }

  Future<void> _localRestore() async {
    try {
      final prefs = await SharedPreferences.getInstance();
      final String? jsonString = prefs.getString(_backupKey);

      if (jsonString == null) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(content: Text(_language == 'ar' ? 'لا توجد نسخة احتياطية سابقة' : 'No previous backup found')),
        );
        return;
      }

      final Map<String, dynamic> backupData = jsonDecode(jsonString);

      // تأكيد من المستخدم قبل مسح البيانات
      bool? confirm = await showDialog<bool>(
        context: context,
        builder: (BuildContext dialogContext) {
          return AlertDialog(
            title: Text(_language == 'ar' ? 'تأكيد الاستعادة' : 'Confirm Restore'),
            content: Text(_language == 'ar'
                ? 'هل أنت متأكد أنك تريد استعادة البيانات؟ هذا سيحل محل البيانات الحالية.'
                : 'Are you sure you want to restore data? This will overwrite current data.'),
            actions: <Widget>[
              TextButton(
                child: Text(_language == 'ar' ? 'إلغاء' : 'Cancel'),
                onPressed: () => Navigator.of(dialogContext).pop(false),
              ),
              TextButton(
                child: Text(_language == 'ar' ? 'استعادة' : 'Restore'),
                onPressed: () => Navigator.of(dialogContext).pop(true),
              ),
            ],
          );
        },
      );

      if (confirm != true) {
        return;
      }

      // مسح البيانات الحالية قبل الاستعادة
      await _database.delete('spending');
      await _database.delete('incomes');
      await _database.delete('loans');
      await _database.delete('budgets');
      await _database.delete('savings_goals');
      await _database.delete('bills');
      await _database.delete('categories');

      // إدراج البيانات المستعادة
      for (var item in (backupData['spending'] as List)) {
        await _database.insert('spending', item);
      }
      for (var item in (backupData['incomes'] as List)) {
        await _database.insert('incomes', item);
      }
      for (var item in (backupData['loans'] as List)) {
        await _database.insert('loans', item);
      }
      for (var item in (backupData['budgets'] as List)) {
        await _database.insert('budgets', item);
      }
      for (var item in (backupData['savings_goals'] as List)) {
        await _database.insert('savings_goals', item);
      }
      for (var item in (backupData['bills'] as List)) {
        await _database.insert('bills', item);
      }
      for (var item in (backupData['categories'] as List)) {
        await _database.insert('categories', item);
      }

      // إعادة جلب البيانات لتحديث واجهة المستخدم
      await _fetchSpendingData();
      await _fetchLoansData();
      await _fetchBudgetsData();
      await _fetchIncomesData();
      await _fetchSavingsGoalsData();
      await _fetchBillsData();
      await _fetchCateg
