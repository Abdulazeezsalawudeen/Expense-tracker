# Expense-tracker
import 'dart:convert';

import 'package:flutter/material.dart';
import 'package:shared_preferences/shared_preferences.dart';

void main() {
  WidgetsFlutterBinding.ensureInitialized();
  runApp(const ExpenseTrackerApp());
}

class ExpenseTrackerApp extends StatelessWidget {
  const ExpenseTrackerApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      debugShowCheckedModeBanner: false,
      title: 'Expense Tracker',
      theme: ThemeData(
        useMaterial3: true,
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.green),
        scaffoldBackgroundColor: const Color(0xFFF5F7F5),
      ),
      home: const ExpenseHomepage(),
    );
  }
}

class Expense {
  final String id;
  final String title;
  final double amount;
  final String category;
  final DateTime date;

  const Expense({
    required this.id,
    required this.title,
    required this.amount,
    required this.category,
    required this.date,
  });

  Map<String, dynamic> toJson() {
    return {
      'id': id,
      'title': title,
      'amount': amount,
      'category': category,
      'date': date.toIso8601String(),
    };
  }

  factory Expense.fromJson(Map<String, dynamic> json) {
    return Expense(
      id: json['id'].toString(),
      title: json['title'].toString(),
      amount: (json['amount'] as num).toDouble(),
      category: json['category'].toString(),
      date: DateTime.parse(json['date'].toString()),
    );
  }
}

// FIX: Controllers are disposed when the dialog widget is removed,
// not immediately when the dialog starts closing.
class DialogControllerDisposer extends StatefulWidget {
  final List<TextEditingController> controllers;
  final Widget child;

  const DialogControllerDisposer({
    super.key,
    required this.controllers,
    required this.child,
  });

  @override
  State<DialogControllerDisposer> createState() =>
      _DialogControllerDisposerState();
}

class _DialogControllerDisposerState extends State<DialogControllerDisposer> {
  @override
  void dispose() {
    for (final controller in widget.controllers) {
      controller.dispose();
    }

    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return widget.child;
  }
}

class ExpenseHomepage extends StatefulWidget {
  const ExpenseHomepage({super.key});

  @override
  State<ExpenseHomepage> createState() => _ExpenseHomepageState();
}

class _ExpenseHomepageState extends State<ExpenseHomepage> {
  static const String _storageKey = 'expense_tracker_data_v1';
  static const String _legacyStorageKey = 'expense_tracker_data';

  SharedPreferences? _preferences;
  Future<bool> _saveQueue = Future<bool>.value(true);

  final List<Expense> _expenses = [];

  double _budget = 50000;
  bool _isLoading = true;
  bool _isSaving = false;
  bool _storageReady = false;

  final List<String> _categories = [
    'Food',
    'Transport',
    'Shopping',
    'Bills',
    'Education',
    'Health',
    'Entertainment',
    'Other',
  ];

  @override
  void initState() {
    super.initState();
    _loadData();
  }

  // LOAD SAVED DATA WHEN THE APP OPENS.
  Future<void> _loadData() async {
    try {
      final preferences = await SharedPreferences.getInstance();
      _preferences = preferences; 
      await preferences.reload();


      final savedData =
          preferences.getString(_storageKey) ??
          preferences.getString(_legacyStorageKey);

      if (savedData != null && savedData.isNotEmpty) {
        final decoded = jsonDecode(savedData);

        if (decoded is Map<String, dynamic>) {
          final savedBudget = decoded['budget'] ?? decoded['monthlyBudget'];
          final savedExpenses = decoded['expenses'];

          double loadedBudget = 50000;
          final List<Expense> loadedExpenses = [];

          if (savedBudget is num && savedBudget.isFinite && savedBudget >= 0) {
            loadedBudget = savedBudget.toDouble();
          }

          if (savedExpenses is List) {
            for (final item in savedExpenses) {
              try {
                if (item is Map<String, dynamic>) {
                  final expense = Expense.fromJson(item);

                  if (expense.title.trim().isNotEmpty &&
                      expense.amount.isFinite &&
                      expense.amount >= 0 &&
                      _categories.contains(expense.category)) {
                    loadedExpenses.add(expense);
                  }
                }
              } catch (error) {
                debugPrint('Skipping invalid expense: $error');
              }
            }
          }

          if (mounted) {
            setState(() {
              _budget = loadedBudget;
              _expenses
                ..clear()
                ..addAll(loadedExpenses);
            });
          }

          debugPrint('DATA LOAD SUCCESS');
          debugPrint('Expenses loaded: ${loadedExpenses.length}');
        }
      } else {
        debugPrint('No saved data found. Starting with default values.');
      }

      _storageReady = true;
    } catch (error, stackTrace) {
      debugPrint('DATA LOAD ERROR: $error');
      debugPrintStack(stackTrace: stackTrace);
    } finally {
      if (mounted) {
        setState(() {
          _isLoading = false;
        });
      } else {
        _isLoading = false;
      }
    }
  }

  // SAVE THE BUDGET AND ALL EXPENSES.
  Future<bool> _saveData() {
    if (!_storageReady || _preferences == null) {
      debugPrint('SAVE SKIPPED: Storage is not ready.');
      _showSaveError();
      return Future<bool>.value(false);
    }

    final save = _saveQueue.then((_) => _writeData());
    _saveQueue = save;
    return save;
  }

  Future<bool> _writeData() async {
    _isSaving = true;

    try {
      final data = {
        'version': 1,
        'budget': _budget,
        'expenses': _expenses.map((expense) => expense.toJson()).toList(),
      };

      final encodedData = jsonEncode(data);
      final preferences = _preferences!;
      final success = await preferences.setString(_storageKey, encodedData);

      if (!success) {
        throw Exception('The device did not confirm the data was saved.');
      }

      final savedData = preferences.getString(_storageKey);

      if (savedData != encodedData) {
        throw Exception('Saved data could not be verified.');
      }

      debugPrint('SAVE SUCCESS');
      return true;
    } catch (error, stackTrace) {
      debugPrint('DATA SAVE ERROR: $error');
      debugPrintStack(stackTrace: stackTrace);

      _showSaveError();
      return false;
    } finally {
      _isSaving = false;
    }
  }

  void _showSaveError() {
    if (!mounted) return;

    ScaffoldMessenger.of(context).showSnackBar(
      const SnackBar(
        content: Text('Could not save your data. Please try again.'),
        backgroundColor: Colors.red,
      ),
    );
  }

  double get _totalSpent {
    return _expenses.fold(0, (total, expense) => total + expense.amount);
  }

  double get _remainingBudget => _budget - _totalSpent;

  String _formatMoney(double amount) {
    return '₦${amount.toStringAsFixed(2)}';
  }

  String _formatDate(DateTime date) {
    final day = date.day.toString().padLeft(2, '0');
    final month = date.month.toString().padLeft(2, '0');
    final year = date.year.toString();

    return '$day/$month/$year';
  }

  IconData _categoryIcon(String category) {
    switch (category) {
      case 'Food':
        return Icons.restaurant;
      case 'Transport':
        return Icons.directions_bus;
      case 'Shopping':
        return Icons.shopping_bag;
      case 'Bills':
        return Icons.receipt_long;
      case 'Education':
        return Icons.school;
      case 'Health':
        return Icons.local_hospital;
      case 'Entertainment':
        return Icons.movie;
      default:
        return Icons.category;
    }
  }

  // ADD A NEW EXPENSE OR EDIT AN EXISTING ONE.
  Future<void> _showExpenseDialog({Expense? existingExpense}) async {
    final isEditing = existingExpense != null;

    final titleController = TextEditingController(
      text: existingExpense?.title ?? '',
    );

    final amountController = TextEditingController(
      text: existingExpense?.amount.toString() ?? '',
    );

    String selectedCategory = existingExpense?.category ?? _categories.first;

    DateTime selectedDate = existingExpense?.date ?? DateTime.now();

    final formKey = GlobalKey<FormState>();

    await showDialog<void>(
      context: context,
      builder: (dialogContext) {
        return StatefulBuilder(
          builder: (context, setDialogState) {
            return DialogControllerDisposer(
              controllers: [titleController, amountController],
              child: AlertDialog(
                title: Text(isEditing ? 'Edit Expense' : 'Add Expense'),
                content: SingleChildScrollView(
                  child: Form(
                    key: formKey,
                    child: Column(
                      mainAxisSize: MainAxisSize.min,
                      children: [
                        TextFormField(
                          controller: titleController,
                          decoration: const InputDecoration(
                            labelText: 'Expense title',
                            hintText: 'e.g. Lunch',
                            prefixIcon: Icon(Icons.edit),
                            border: OutlineInputBorder(),
                          ),
                          validator: (value) {
                            if (value == null || value.trim().isEmpty) {
                              return 'Please enter an expense title.';
                            }

                            return null;
                          },
                        ),
                        const SizedBox(height: 16),
                        TextFormField(
                          controller: amountController,
                          keyboardType: const TextInputType.numberWithOptions(
                            decimal: true,
                          ),
                          decoration: const InputDecoration(
                            labelText: 'Amount (₦)',
                            hintText: 'e.g. 2000',
                            prefixIcon: Icon(Icons.payments),
                            border: OutlineInputBorder(),
                          ),
                          validator: (value) {
                            final amount = double.tryParse(value?.trim() ?? '');

                            if (amount == null ||
                                !amount.isFinite ||
                                amount <= 0) {
                              return 'Enter an amount greater than zero.';
                            }

                            return null;
                          },
                        ),
                        const SizedBox(height: 16),
                        DropdownButtonFormField<String>(
                          initialValue: selectedCategory,
                          decoration: const InputDecoration(
                            labelText: 'Category',
                            prefixIcon: Icon(Icons.category),
                            border: OutlineInputBorder(),
                          ),
                          items: _categories.map((category) {
                            return DropdownMenuItem<String>(
                              value: category,
                              child: Text(category),
                            );
                          }).toList(),
                          onChanged: (value) {
                            if (value != null) {
                              setDialogState(() {
                                selectedCategory = value;
                              });
                            }
                          },
                        ),
                        const SizedBox(height: 16),
                        ListTile(
                          contentPadding: EdgeInsets.zero,
                          leading: const Icon(Icons.calendar_month),
                          title: const Text('Expense date'),
                          subtitle: Text(_formatDate(selectedDate)),
                          trailing: IconButton(
                            icon: const Icon(Icons.edit_calendar),
                            onPressed: () async {
                              final pickedDate = await showDatePicker(
                                context: context,
                                initialDate: selectedDate,
                                firstDate: DateTime(2000),
                                lastDate: DateTime.now(),
                              );

                              if (pickedDate != null) {
                                setDialogState(() {
                                  selectedDate = pickedDate;
                                });
                              }
                            },
                          ),
                        ),
                      ],
                    ),
                  ),
                ),
                actions: [
                  TextButton(
                    onPressed: () {
                      Navigator.of(dialogContext).pop();
                    },
                    child: const Text('Cancel'),
                  ),
                  FilledButton(
                    onPressed: () async {
                      if (_isSaving) return;

                      if (!formKey.currentState!.validate()) {
                        return;
                      }

                      final title = titleController.text.trim();
                      final amount = double.parse(amountController.text.trim());

                      final expense = Expense(
                        id:
                            existingExpense?.id ??
                            DateTime.now().microsecondsSinceEpoch.toString(),
                        title: title,
                        amount: amount,
                        category: selectedCategory,
                        date: selectedDate,
                      );

                      setState(() {
                        if (isEditing) {
                          final index = _expenses.indexWhere(
                            (item) => item.id == existingExpense.id,
                          );

                          if (index != -1) {
                            _expenses[index] = expense;
                          }
                        } else {
                          _expenses.insert(0, expense);
                        }
                      });

                      Navigator.of(dialogContext).pop();

                      final saved = await _saveData();

                      if (mounted && saved) {
                        ScaffoldMessenger.of(context).showSnackBar(
                          SnackBar(
                            content: Text(
                              isEditing ? 'Expense updated.' : 'Expense added.',
                            ),
                          ),
                        );
                      }
                    },
                    child: Text(isEditing ? 'Update' : 'Save'),
                  ),
                ],
              ),
            );
          },
        );
      },
    );

    // FIX: Do not dispose the controllers here.
    // DialogControllerDisposer handles their disposal safely.
  }

  // CHANGE AND SAVE THE MONTHLY BUDGET.
  Future<void> _showBudgetDialog() async {
    final controller = TextEditingController(text: _budget.toStringAsFixed(2));

    final formKey = GlobalKey<FormState>();

    await showDialog<void>(
      context: context,
      builder: (dialogContext) {
        return DialogControllerDisposer(
          controllers: [controller],
          child: AlertDialog(
            title: const Text('Set Your Budget'),
            content: Form(
              key: formKey,
              child: TextFormField(
                controller: controller,
                autofocus: true,
                keyboardType: const TextInputType.numberWithOptions(
                  decimal: true,
                ),
                decoration: const InputDecoration(
                  labelText: 'Budget amount (₦)',
                  prefixIcon: Icon(Icons.account_balance_wallet),
                  border: OutlineInputBorder(),
                ),
                validator: (value) {
                  final amount = double.tryParse(value?.trim() ?? '');

                  if (amount == null || !amount.isFinite || amount < 0) {
                    return 'Enter a valid budget amount.';
                  }

                  return null;
                },
              ),
            ),
            actions: [
              TextButton(
                onPressed: () {
                  Navigator.of(dialogContext).pop();
                },
                child: const Text('Cancel'),
              ),
              FilledButton(
                onPressed: () async {
                  if (_isSaving) return;

                  if (!formKey.currentState!.validate()) {
                    return;
                  }

                  final newBudget = double.parse(controller.text.trim());

                  setState(() {
                    _budget = newBudget;
                  });

                  Navigator.of(dialogContext).pop();

                  final saved = await _saveData();

                  if (mounted && saved) {
                    ScaffoldMessenger.of(context).showSnackBar(
                      const SnackBar(content: Text('Budget saved.')),
                    );
                  }
                },
                child: const Text('Save Budget'),
              ),
            ],
          ),
        );
      },
    );

    // FIX: The dialog widget now handles controller disposal.
  }

  // DELETE AN EXPENSE.
  Future<void> _deleteExpense(Expense expense) async {
    final confirm = await showDialog<bool>(
      context: context,
      builder: (dialogContext) {
        return AlertDialog(
          title: const Text('Delete Expense?'),
          content: Text('Are you sure you want to delete "${expense.title}"?'),
          actions: [
            TextButton(
              onPressed: () {
                Navigator.of(dialogContext).pop(false);
              },
              child: const Text('Cancel'),
            ),
            FilledButton(
              style: FilledButton.styleFrom(backgroundColor: Colors.red),
              onPressed: () {
                Navigator.of(dialogContext).pop(true);
              },
              child: const Text('Delete'),
            ),
          ],
        );
      },
    );

    if (confirm != true || !mounted) return;

    setState(() {
      _expenses.removeWhere((item) => item.id == expense.id);
    });

    final saved = await _saveData();

    if (mounted && saved) {
      ScaffoldMessenger.of(context)
          .showSnackBar(const SnackBar(content: Text('Expense deleted.')));
    }
  }

  Widget _buildSummaryCard({
    required String title,
    required String amount,
    required IconData icon,
    required Color color,
  }) {
    return Card(
      elevation: 0,
      color: color.withValues(alpha: 0.10),
      child: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Icon(icon, color: color, size: 28),
            const SizedBox(height: 12),
            Text(
              title,
              style: const TextStyle(fontSize: 13, color: Colors.black54),
            ),
            const SizedBox(height: 6),
            FittedBox(
              fit: BoxFit.scaleDown,
              alignment: Alignment.centerLeft,
              child: Text(
                amount,
                style: TextStyle(
                  fontSize: 19,
                  fontWeight: FontWeight.bold,
                  color: color,
                ),
              ),
            ),
          ],
        ),
      ),
    );
  }

  Widget _buildExpenseTile(Expense expense) {
    return Card(
      margin: const EdgeInsets.only(bottom: 10),
      elevation: 0,
      color: Colors.white,
      child: ListTile(
        contentPadding: const EdgeInsets.symmetric(horizontal: 14, vertical: 6),
        leading: CircleAvatar(
          backgroundColor: Colors.green.withValues(alpha: 0.12),
          child: Icon(
            _categoryIcon(expense.category),
            color: Colors.green.shade800,
          ),
        ),
        title: Text(
          expense.title,
          style: const TextStyle(fontWeight: FontWeight.w600),
        ),
        subtitle: Text('${expense.category} • ${_formatDate(expense.date)}'),
        trailing: PopupMenuButton<String>(
          onSelected: (value) {
            if (value == 'edit') {
              _showExpenseDialog(existingExpense: expense);
            } else if (value == 'delete') {
              _deleteExpense(expense);
            }
          },
          itemBuilder: (context) => const [
            PopupMenuItem(
              value: 'edit',
              child: Row(
                children: [
                  Icon(Icons.edit, size: 20),
                  SizedBox(width: 8),
                  Text('Edit'),
                ],
              ),
            ),
            PopupMenuItem(
              value: 'delete',
              child: Row(
                children: [
                  Icon(Icons.delete, color: Colors.red, size: 20),
                  SizedBox(width: 8),
                  Text('Delete'),
                ],
              ),
            ),
          ],
          child: Column(
            mainAxisAlignment: MainAxisAlignment.center,
            crossAxisAlignment: CrossAxisAlignment.end,
            children: [
              Text(
                _formatMoney(expense.amount),
                style: const TextStyle(
                  fontWeight: FontWeight.bold,
                  fontSize: 14,
                ),
              ),
              const SizedBox(height: 4),
              const Icon(Icons.more_horiz, color: Colors.grey),
            ],
          ),
        ),
      ),
    );
  }

  @override
  Widget build(BuildContext context) {
    final spent = _totalSpent;
    final remaining = _remainingBudget;

    final spendingProgress = _budget > 0
        ? (spent / _budget).clamp(0.0, 1.0)
        : (spent > 0 ? 1.0 : 0.0);

    return Scaffold(
      appBar: AppBar(
        title: const Text(
          'Expense Tracker',
          style: TextStyle(fontWeight: FontWeight.bold),
        ),
        backgroundColor: Colors.green.shade700,
        foregroundColor: Colors.white,
        actions: [
          IconButton(
            tooltip: 'Set budget',
            onPressed: _isLoading ? null : _showBudgetDialog,
            icon: const Icon(Icons.account_balance_wallet),
          ),
        ],
      ),
      body: _isLoading
          ? const Center(
              child: Column(
                mainAxisSize: MainAxisSize.min,
                children: [
                  CircularProgressIndicator(),
                  SizedBox(height: 16),
                  Text('Loading your saved data...'),
                ],
              ),
            )
          : SafeArea(
              child: ListView(
                padding: const EdgeInsets.all(16),
                children: [
                  Container(
                    padding: const EdgeInsets.all(20),
                    decoration: BoxDecoration(
                      color: Colors.green.shade700,
                      borderRadius: BorderRadius.circular(20),
                    ),
                    child: Column(
                      crossAxisAlignment: CrossAxisAlignment.start,
                      children: [
                        const Text(
                          'Monthly Budget',
                          style: TextStyle(color: Colors.white70, fontSize: 15),
                        ),
                        const SizedBox(height: 8),
                        Text(
                          _formatMoney(_budget),
                          style: const TextStyle(
                            color: Colors.white,
                            fontSize: 30,
                            fontWeight: FontWeight.bold,
                          ),
                        ),
                        const SizedBox(height: 20),
                        Row(
                          mainAxisAlignment: MainAxisAlignment.spaceBetween,
                          children: [
                            const Text(
                              'Budget used',
                              style: TextStyle(color: Colors.white),
                            ),
                            Text(
                              '${(spendingProgress * 100).toStringAsFixed(1)}%',
                              style: const TextStyle(
                                color: Colors.white,
                                fontWeight: FontWeight.bold,
                              ),
                            ),
                          ],
                        ),
                        const SizedBox(height: 8),
                        ClipRRect(
                          borderRadius: BorderRadius.circular(10),
                          child: LinearProgressIndicator(
                            value: spendingProgress,
                            minHeight: 9,
                            backgroundColor: Colors.white24,
                            valueColor: const AlwaysStoppedAnimation<Color>(
                              Colors.white,
                            ),
                          ),
                        ),
                        const SizedBox(height: 12),
                        Align(
                          alignment: Alignment.centerRight,
                          child: TextButton.icon(
                            onPressed: _showBudgetDialog,
                            style: TextButton.styleFrom(
                              foregroundColor: Colors.white,
                            ),
                            icon: const Icon(Icons.edit, size: 18),
                            label: const Text('Edit Budget'),
                          ),
                        ),
                      ],
                    ),
                  ),
                  const SizedBox(height: 12),
                  Row(
                    children: [
                      Expanded(
                        child: _buildSummaryCard(
                          title: 'Total Spent',
                          amount: _formatMoney(spent),
                          icon: Icons.trending_down,
                          color: Colors.orange.shade800,
                        ),
                      ),
                      Expanded(
                        child: _buildSummaryCard(
                          title: 'Remaining',
                          amount: _formatMoney(remaining),
                          icon: Icons.account_balance_wallet,
                          color: remaining < 0
                              ? Colors.red
                              : Colors.green.shade800,
                        ),
                      ),
                    ],
                  ),
                  const SizedBox(height: 24),
                  Row(
                    mainAxisAlignment: MainAxisAlignment.spaceBetween,
                    children: [
                      const Text(
                        'Recent Expenses',
                        style: TextStyle(
                          fontSize: 20,
                          fontWeight: FontWeight.bold,
                        ),
                      ),
                      Text(
                        '${_expenses.length} item${_expenses.length == 1 ? '' : 's'}',
                        style: const TextStyle(color: Colors.black54),
                      ),
                    ],
                  ),
                  const SizedBox(height: 12),
                  if (_expenses.isEmpty)
                    Container(
                      padding: const EdgeInsets.all(30),
                      decoration: BoxDecoration(
                        color: Colors.white,
                        borderRadius: BorderRadius.circular(16),
                      ),
                      child: const Column(
                        children: [
                          Icon(
                            Icons.receipt_long,
                            size: 55,
                            color: Colors.grey,
                          ),
                          SizedBox(height: 12),
                          Text(
                            'No expenses yet',
                            style: TextStyle(
                              fontSize: 17,
                              fontWeight: FontWeight.bold,
                            ),
                          ),
                          SizedBox(height: 6),
                          Text(
                            'Tap the + button to record your first expense.',
                            textAlign: TextAlign.center,
                            style: TextStyle(color: Colors.black54),
                          ),
                        ],
                      ),
                    )
                  else
                    ..._expenses.map(_buildExpenseTile),
                  const SizedBox(height: 80),
                ],
              ),
            ),
      floatingActionButton: FloatingActionButton.extended(
        onPressed: _isLoading ? null : () => _showExpenseDialog(),
        backgroundColor: Colors.green.shade700,
        foregroundColor: Colors.white,
        icon: const Icon(Icons.add),
        label: const Text('Add Expense'),
      ),
    );
  }
}
