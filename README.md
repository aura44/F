import 'dart:async';
import 'dart:typed_data';
import 'package:flutter/material.dart';
import 'package:qr_code_scanner/qr_code_scanner.dart';
import 'package:qr_flutter/qr_flutter.dart';
import 'package:in_app_purchase/in_app_purchase.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  final bool isAvailable = await InAppPurchaseConnection.instance.isAvailable();
  if (isAvailable) {
    // Initialize the in-app purchase system
    await InAppPurchaseConnection.instance.restorePurchases();
  }
  runApp(MyApp());
}

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'QR Code Scanner',
      theme: ThemeData(
        primarySwatch: Colors.blue,
        accentColor: Colors.deepOrange,
        fontFamily: 'Open Sans',
        textTheme: ThemeData.light().textTheme.copyWith(
              headline6: TextStyle(
                fontFamily: 'Open Sans',
                fontSize: 24,
                fontWeight: FontWeight.bold,
              ),
              subtitle1: TextStyle(
                fontFamily: 'Open Sans',
                fontSize: 18,
                fontWeight: FontWeight.bold,
              ),
            ),
        elevatedButtonTheme: ElevatedButtonThemeData(
          style: ElevatedButton.styleFrom(
            primary: Colors.deepOrange,
            onPrimary: Colors.white,
            textStyle: TextStyle(
              fontFamily: 'Open Sans',
              fontSize: 18,
              fontWeight: FontWeight.bold,
            ),
            padding: EdgeInsets.symmetric(
              vertical: 12,
              horizontal: 24,
            ),
            shape: RoundedRectangleBorder(
              borderRadius: BorderRadius.circular(20),
            ),
          ),
        ),
      ),
      home: MyHomePage(title: 'QR Code Scanner'),
    );
  }
}

class MyHomePage extends StatefulWidget {
  MyHomePage({Key? key, required this.title}) : super(key: key);

  final String title;

  @override
  _MyHomePageState createState() => _MyHomePageState();
}

class _MyHomePageState extends State<MyHomePage> {
  final GlobalKey qrKey = GlobalKey(debugLabel: 'QR');
  late QRViewController controller;
  bool showQR = false;

  @override
  void dispose() {
    controller.dispose();
    super.dispose();
  }

  void _onQRViewCreated(QRViewController controller) {
    setState(() {
      this.controller = controller;
    });
    controller.scannedDataStream.listen((scanData) async {
      _showSnackBar(scanData.code);
      if (await _isPremiumUser()) {
        // Generate QR code for scanned data
        _generateQR(scanData.code);
      } else {
        // Prompt user to purchase premium to generate QR codes
        _promptPurchase();
      }
    });
  }

  void _handleManualInput() {
    showDialog(
      context: context,
      builder: (BuildContext context) {
        String input = '';
        return AlertDialog(
          title: Text('Enter QR Code'),
          content: TextFormField(
            onChanged: (value) {
              input = value;
            },
          ),
          actions: <Widget>[
            TextButton(
              child: Text('CANCEL'),
              onPressed: () {
                Navigator.of(context).pop();
              },
            ),
            TextButton(
              child: Text('OK'),
              onPressed: () async {
                if (await _isPremiumUser()) {
                  // Generate QR code for manual input
                  _generateQR(input);
                  Navigator.of(context).pop();
                } else {
                  // Prompt user to purchase premium to generate QR codes
                  _promptPurchase();
                }
              },
            ),
          ],
        );
      },
    );
  }

  Future<void
