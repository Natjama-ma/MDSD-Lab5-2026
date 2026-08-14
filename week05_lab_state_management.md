# ใบงานปฏิบัติสัปดาห์ที่ 5: State Management ด้วย Provider และ Riverpod

**วิชา** การพัฒนาซอฟต์แวร์สำหรับอุปกรณ์เคลื่อนที่ | **เครื่องมือ** Flutter, Provider, Riverpod, Google AI Studio (Gemini API)


---

## วัตถุประสงค์การเรียนรู้

เมื่อทำใบงานนี้เสร็จสิ้น ผู้เรียนจะสามารถ

1. สร้าง Global State ด้วย `ChangeNotifier` และ `ChangeNotifierProvider` ให้หลายหน้าจอใช้ข้อมูลร่วมกันได้ถูกต้อง
2. แยกแยะและใช้งาน `context.watch()` กับ `context.read()` ได้ตรงตามสถานการณ์
3. รีแฟกเตอร์โค้ดที่เขียนด้วย `setState` แบบมี Prop Drilling ให้กลายเป็นโครงสร้างที่ใช้ Provider
4. ทดลองสร้าง Provider แบบเดียวกันด้วย Riverpod และเปรียบเทียบความแตกต่างของโค้ดจริง
5. ใช้ Google AI Studio (Gemini) ช่วยวิเคราะห์และให้เหตุผลในการเลือกเครื่องมือ State Management ให้เหมาะกับโจทย์
6. ออกแบบและตัดสินใจเลือกวิธีจัดการ State ด้วยตนเองสำหรับฟีเจอร์ใหม่ พร้อมทั้งอธิบายเหตุผลของการเลือกนั้นได้

## สิ่งที่ต้องเตรียมก่อนเริ่ม

- ติดตั้ง Flutter SDK และ VS Code เรียบร้อยจากสัปดาห์ที่ 1
- โปรเจกต์ Flutter `campus_marketplace` จากสัปดาห์ที่ 4 ที่มี Multi-screen Navigation ด้วย Go Router (หากยังไม่มี ให้สร้างโปรเจกต์ใหม่ชื่อ `campus_marketplace` ด้วยคำสั่ง `flutter create campus_marketplace`)
- บัญชี Google AI Studio ที่สร้างไว้ตั้งแต่สัปดาห์ที่ 1

---

## ทบทวนทฤษฎีก่อนเริ่มลงมือปฏิบัติ

ก่อนลงมือเขียนโค้ด ให้ทบทวนแนวคิดสำคัญต่อไปนี้ให้แม่นยำ เพราะทุกขั้นตอนในใบงานนี้อ้างอิงจากหลักการเหล่านี้โดยตรง 

### State สองชนิด: Ephemeral State กับ App State 

**Ephemeral State** คือข้อมูลที่มีความหมายเฉพาะภายใน Widget เดียวหรือกลุ่มเล็ก ๆ ที่อยู่ใกล้กัน ไม่มี Widget อื่นในแอปจำเป็นต้องรู้ (เช่น ค่าที่พิมพ์ค้างใน TextField, ตำแหน่งแท็บที่เลือกอยู่) เปรียบเหมือนโน้ตกระดาษบนโต๊ะทำงานของตัวเอง ใช้ `setState` จัดการก็เพียงพอ

**App State** คือข้อมูลที่ต้องใช้ร่วมกันโดยหลาย Widget ที่อาจอยู่คนละกิ่งของ Widget Tree หรือคนละ Route (เช่น สถานะล็อกอิน, รายการโปรด, ธีมสี) เปรียบเหมือนกระดานประกาศกลางที่ทุกแผนกต้องเห็นตรงกัน ต้องใช้ InheritedWidget, Provider หรือ Riverpod จัดการ

| มิติ | Ephemeral State | App State |
|---|---|---|
| ขอบเขตการมองเห็น | Widget เดียวหรือกลุ่มเล็ก ๆ | หลายหน้าจอ / ทั้งแอป |
| อายุการใช้งาน | สั้น มักหายเมื่อ Widget dispose | ยาว อยู่ตลอดการใช้งานแอป |
| เครื่องมือที่เหมาะสม | `setState` ภายใน StatefulWidget | InheritedWidget, Provider, Riverpod |

ในใบงานนี้ **รายการสินค้าที่บันทึกไว้ (Favorites)** คือตัวอย่างของ App State ชัดเจน เพราะต้องแสดงผลตรงกันทั้งที่ AppBar ของหน้า Home และในหน้า Favorites ที่แยก Route ออกไป — นี่คือเหตุผลที่ส่วนที่ 1 ของใบงานจะทำให้เห็นปัญหาก่อน แล้วส่วนที่ 2 จึงแก้ด้วย Provider

### Prop Drilling คือปัญหาอะไร 

เมื่อใช้ `setState` เก็บ App State ไว้ที่ Widget ต้นทาง (เช่น `HomePage`) แล้วต้องส่งค่า/ฟังก์ชันลงไปให้ Widget ลูกหลานที่อยู่ลึกหลายชั้นผ่าน constructor ทีละชั้น ทั้งที่ Widget ตัวกลางไม่ได้ใช้ค่าเหล่านั้นเองเลย เรียกปรากฏการณ์นี้ว่า **Prop Drilling** ยิ่ง Widget ตัวกลางเพิ่มขึ้น ยิ่งต้องแก้ constructor ซ้ำไปเรื่อย ๆ และปัญหาจะรุนแรงขึ้นไปอีกเมื่อต้องส่งข้อมูลข้าม Route (เพราะ constructor ส่งได้เฉพาะพ่อแม่ลูกในทรีเดียวกันเท่านั้น) — ส่วนที่ 1 ของใบงานนี้จะให้ลงมือสร้างปัญหานี้ด้วยตัวเองก่อน เพื่อให้เห็นภาพว่า Provider เข้ามาแก้อะไร

### หลักการของ Provider 

Provider คือแพ็กเกจที่ห่อหุ้ม `InheritedWidget` (กลไกกระจายข้อมูลลงทรีของ Flutter เอง) ให้ใช้งานง่ายขึ้น โดยผนวกกับ **ChangeNotifier** ซึ่งมีเมธอด `notifyListeners()` ทำหน้าที่เหมือน `setState` แต่ขยายขอบเขตจาก "Widget เดียว" เป็น "ผู้ฟังจำนวนเท่าใดก็ได้ทั่วทั้งแอป" หลักการสำคัญที่ต้องจำให้แม่นคือ

- **กฎทอง**: ทุกเมธอดที่แก้ไขข้อมูลใน Model ต้องเรียก `notifyListeners()` เสมอ ไม่งั้น UI จะไม่อัปเดต
- **`context.watch<T>()`**: อ่านค่าและสมัครเป็นผู้ติดตาม จะถูก rebuild ทุกครั้งที่ข้อมูลเปลี่ยน ใช้กับส่วนที่ต้อง *แสดงผล*
- **`context.read<T>()`**: อ่านค่าครั้งเดียวเพื่อเรียกเมธอด ไม่สมัครเป็นผู้ติดตาม ใช้กับการกดปุ่ม (`onPressed`) เพื่อไม่ให้ Widget นั้น rebuild โดยไม่จำเป็น

### หลักการของ Riverpod 

Riverpod คือวิวัฒนาการของ Provider ที่แก้ข้อจำกัดเรื่องการต้องพึ่งพา `BuildContext` โดยใช้ `WidgetRef` แทน (`ref.watch(...)` เทียบเท่า `context.watch<T>()` และ `ref.read(...)` เทียบเท่า `context.read<T>()`) และเพิ่มความปลอดภัยด้าน Type ตั้งแต่ตอนเขียนโค้ด (compile-time) แทนที่จะพังตอนรันจริง แนวคิดหลักเหมือนกับ Provider ทุกประการ เปลี่ยนแค่วิธีเข้าถึงข้อมูล

### กรอบการตัดสินใจเลือกเครื่องมือ

เริ่มจาก `setState` เสมอถ้าข้อมูลอยู่ในขอบเขต Widget เดียว เมื่อข้อมูลต้องใช้ข้ามหลายหน้าจอให้ยกระดับไปใช้ Provider ก่อน (เรียนรู้ง่ายกว่า) และเมื่อโปรเจกต์ต้องการ Unit Test ที่เข้มงวดหรือ Type Safety สูงขึ้น จึงค่อยพิจารณาย้ายไปใช้ Riverpod ใบงานนี้จะพาทำครบทั้งสามระดับตามลำดับนี้เอง

---

## ส่วนที่ 1: สร้างปัญหาให้เกิดขึ้น ก่อนแก้ไข (Prop Drilling Demo)

ก่อนใช้ Provider เราจะสร้างสถานการณ์ปัญหาขึ้นมาก่อนด้วยตัวเอง เพื่อให้เห็นภาพว่า Provider แก้ปัญหาอะไร ให้ทำตามทีละขั้นตอนต่อไปนี้ให้ครบ หลังจากนั้นจะได้แอปที่รันได้จริงและเห็นปัญหา Prop Drilling ชัดเจน

### ขั้นตอนที่ 1.1: สร้างโมเดล Item
สร้างโปรเจกต์ใหม่ชื่อ `campus_marketplace` ด้วยคำสั่ง `flutter create campus_marketplace` หลังจากนั้น
สร้างไฟล์ `lib/models/item.dart` และเพิ่มโค้ดต่อไปนี้ — นี่คือจุดเริ่มต้นของโมเดล `Item` ที่จะเป็นแกนกลางของแอป Campus Marketplace ตลอดทั้งเทอม 

```dart
class Item {
  final String id;
  final String title;
  final double price;

  const Item({required this.id, required this.title, required this.price});
}

// ข้อมูลจำลอง (mock) ไว้ใช้ก่อน สัปดาห์ที่ 6 จะเปลี่ยนมาดึงจาก API จริงแทนลิสต์นี้
final catalog = <Item>[
  const Item(id: 'i1', title: 'หนังสือ Calculus มือสอง', price: 150),
  const Item(id: 'i2', title: 'หูฟังไร้สาย (สภาพดี 90%)', price: 450),
  const Item(id: 'i3', title: 'โคมไฟตั้งโต๊ะหอพัก', price: 120),
];
```

### ขั้นตอนที่ 1.2: สร้าง ItemCard (Widget ชั้นในสุด)

สร้างไฟล์ `lib/widgets/item_card.dart` — Widget นี้อยู่ชั้นล่างสุดของทรี เป็นตัวที่ **ใช้งานจริง** ทั้ง `savedItems` (เพื่อเช็คว่าไอเทมนี้ถูกบันทึกไปแล้วหรือยัง) และ `onSave` (เพื่อเรียกตอนกดปุ่ม)

```dart
import 'package:flutter/material.dart';
import '../models/item.dart';

class ItemCard extends StatelessWidget {
  final Item item;
  final List<Item> savedItems; // ต้องรับมาเพื่อเช็คว่าไอเทมนี้ถูกบันทึกแล้วหรือยัง (Prop Drilling)
  final void Function(Item item) onSave; // ฟังก์ชันที่ถูกส่งทอดมาจาก HomePage ผ่าน ItemListSection

  const ItemCard({
    super.key,
    required this.item,
    required this.savedItems,
    required this.onSave,
  });

  @override
  Widget build(BuildContext context) {
    // เช็คว่าไอเทมนี้ถูกบันทึกไปแล้วหรือยัง โดยเทียบ id กับรายการที่ส่งเข้ามา
    final alreadySaved = savedItems.any((i) => i.id == item.id);

    return Card(
      margin: const EdgeInsets.symmetric(horizontal: 12, vertical: 6),
      child: ListTile(
        title: Text(item.title),
        subtitle: Text('฿${item.price.toStringAsFixed(0)}'),
        trailing: ElevatedButton(
          // ปิดปุ่ม (onPressed: null) ถ้าบันทึกไปแล้ว ป้องกันการกดซ้ำสร้างรายการซ้ำ
          onPressed: alreadySaved ? null : () => onSave(item),
          child: Text(alreadySaved ? '❤️ บันทึกแล้ว' : '🤍 บันทึกเป็นรายการโปรด'),
        ),
      ),
    );
  }
}
```

### ขั้นตอนที่ 1.3: สร้าง ItemListSection (Widget ชั้นกลาง — จุดที่เกิด Prop Drilling)

สร้างไฟล์ `lib/widgets/item_list_section.dart` — สังเกตให้ดีว่า Widget นี้ **ไม่ได้ใช้** `savedItems` หรือ `onSave` โดยตรงเลยสักบรรทัด มันแค่รับพารามิเตอร์มาแล้วส่งต่อให้ `ItemCard` แต่ละใบเท่านั้น นี่คือแก่นแท้ของปัญหา Prop Drilling

```dart
import 'package:flutter/material.dart';
import '../models/item.dart';
import 'item_card.dart';

class ItemListSection extends StatelessWidget {
  final List<Item> catalog;
  final List<Item> savedItems; // รับมาจาก HomePage แล้วต้อง "ส่งทอด" ต่อให้ ItemCard ทุกใบ
  final void Function(Item item) onSave; // ฟังก์ชันเดียวกันที่ต้องส่งทอดต่อเช่นกัน

  const ItemListSection({
    super.key,
    required this.catalog,
    required this.savedItems,
    required this.onSave,
  });

  @override
  Widget build(BuildContext context) {
    return ListView.builder(
      shrinkWrap: true,
      physics: const NeverScrollableScrollPhysics(),
      itemCount: catalog.length,
      itemBuilder: (context, index) {
        final item = catalog[index];
        // ตัวมันเองไม่แตะ savedItems/onSave เลย แค่ "ส่งผ่าน" ไปให้ ItemCard เท่านั้น
        return ItemCard(item: item, savedItems: savedItems, onSave: onSave);
      },
    );
  }
}
```

### ขั้นตอนที่ 1.4: สร้าง HomePage (Widget ชั้นบนสุด — เจ้าของ State ตัวจริง)

สร้าง/แก้ไขไฟล์ `lib/home_page.dart` ให้เป็น `StatefulWidget` ที่เก็บรายการสินค้าที่บันทึกไว้ (Favorites) ไว้ในตัวเอง แล้ว "ยัด" ค่าและฟังก์ชันลงไปให้ Widget ลูกทั้งสองชั้นที่สร้างไว้ในขั้นตอนที่ 1.2-1.3

```dart
import 'package:flutter/material.dart';
import 'models/item.dart';
import 'widgets/item_list_section.dart';

class HomePage extends StatefulWidget {
  const HomePage({super.key});

  @override
  State<HomePage> createState() => _HomePageState();
}

class _HomePageState extends State<HomePage> {
  final List<Item> _savedItems = []; // เก็บรายการโปรดไว้ใน State ของ HomePage เอง (ยังไม่ใช้ Provider)

  void _onSave(Item item) {
    setState(() {
      _savedItems.add(item); // แก้ไข List แล้วสั่ง rebuild ทั้งทรีที่อยู่ใต้ HomePage
    });
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Campus Marketplace'),
        actions: [
          Padding(
            padding: const EdgeInsets.only(right: 12),
            child: Center(child: Text('❤️ ${_savedItems.length}')),
          ),
        ],
      ),
      body: ItemListSection(
        catalog: catalog,       // มาจาก item.dart ที่สร้างไว้ในขั้นตอนที่ 1.1
        savedItems: _savedItems, // ต้องส่งลงไปให้ ItemListSection แม้มันไม่ได้ใช้เอง
        onSave: _onSave,         // ส่งฟังก์ชันลงไปเช่นกัน — รวมเป็น "Prop Drilling" 2 ชั้น
      ),
    );
  }
}
```

แก้ไขไฟล์ `lib/main.dart` ให้เป็นดังนี้ เพื่อให้แอปเริ่มทำงานที่ `HomePage` (ยังไม่มี Provider ในขั้นตอนนี้ จะเพิ่มในส่วนที่ 2)

```dart
import 'package:flutter/material.dart';
import 'home_page.dart';

void main() {
  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return const MaterialApp(
      title: 'Campus Marketplace',
      debugShowCheckedModeBanner: false, // ปิดริบบิ้น DEBUG มุมขวาบน ไม่ให้บังไอคอนหัวใจใน AppBar
      home: HomePage(), // เรียก HomePage ที่สร้างไว้ในขั้นตอนที่ 1.4 เป็นหน้าแรกของแอป
    );
  }
}
```

> ✅ **Checkpoint 1.1** รันแอปและกดปุ่ม "🤍 บันทึกเป็นรายการโปรด" ที่สินค้าชิ้นใดก็ได้ ทดสอบว่า (ก) ตัวเลขในไอคอนหัวใจที่ AppBar เพิ่มขึ้นถูกต้อง และ (ข) ปุ่มของสินค้าที่กดไปแล้วเปลี่ยนเป็น "❤️ บันทึกแล้ว" และกดซ้ำไม่ได้ ถ่ายภาพหน้าจอที่เห็นทั้งสองอย่างนี้พร้อมกัน แล้วเปิดไฟล์ `item_card.dart` และ `item_list_section.dart` ให้เห็น constructor ที่ต้องรับพารามิเตอร์ส่งต่อ (Prop Drilling) ชัดเจน แนบส่งในรายงาน


<img width="1919" height="942" alt="image" src="https://github.com/user-attachments/assets/681c1b36-6144-49b4-9452-14b9014c0267" />



<img width="824" height="269" alt="Screenshot 2026-08-14 103914" src="https://github.com/user-attachments/assets/2347e660-7b4e-4a76-bca6-3df439931117" />



<img width="854" height="282" alt="Screenshot 2026-08-14 103934" src="https://github.com/user-attachments/assets/19d7759f-94d8-418b-a424-d840fe37845e" />

**คำถาม**: ถ้าต้องเพิ่มหน้าจอ `FavoritesPage` ที่ต้องแสดงรายการที่บันทึกไว้ชุดเดียวกัน แต่ถูก push แยกออกไปเป็นอีก Route หนึ่ง จะเกิดปัญหาอะไรกับโค้ดแบบ Prop Drilling นี้ จงเขียนคำตอบสั้น ๆ 

```text
ทำให้การซิงก์ข้อมูลข้าม Route จะยากและซับซ้อน เพราะการส่ง State ผ่าน Constructor ข้ามไปยังหน้าจอใหม่ที่ถูก push ขึ้นมาจะยุ่งยาก
และถ้าการแก้ไขข้อมูล (เช่น ลบรายการโปรด) ในหน้า FavoritesPage หน้า HomePage เดิมก็จะไม่รับรู้และไม่อัปเดต UI (Rebuild) อัตโนมัติ

```

---

## ส่วนที่ 2: รีแฟกเตอร์ด้วย Provider

### ขั้นตอนที่ 2.1: ติดตั้งแพ็กเกจ

เพิ่ม dependency ในไฟล์ `pubspec.yaml`

```yaml
dependencies:
  flutter:
    sdk: flutter
  provider: ^6.1.2
```

รันคำสั่งในเทอร์มินัลของ VS Code

```bash
flutter pub get
```

### ขั้นตอนที่ 2.2: สร้าง FavoritesModel

สร้างไฟล์ `lib/models/favorites_model.dart` — นี่คือคลาสที่จะเป็น "Single Source of Truth" ของรายการโปรดทั้งแอป แทนที่การเก็บ State กระจัดกระจายแบบส่วนที่ 1

```dart
import 'package:flutter/foundation.dart';
import 'item.dart';

class FavoritesModel extends ChangeNotifier {
  final List<Item> _items = []; // ตั้งเป็น private (ขึ้นต้นด้วย _) เพื่อไม่ให้ภายนอกแก้ไขตรง ๆ ได้

  List<Item> get items => List.unmodifiable(_items); // เปิดให้อ่านได้ แต่แก้ไขผ่าน list นี้ไม่ได้
  int get itemCount => _items.length;
  double get totalValue => _items.fold(0, (sum, i) => sum + i.price);

  void add(Item item) {
    _items.add(item);
    notifyListeners(); // กฎทองของ ChangeNotifier: แก้ข้อมูลแล้วต้องแจ้งทุกครั้ง ไม่งั้น UI จะไม่อัปเดต
  }

  void remove(Item item) {
    _items.remove(item);
    notifyListeners();
  }

  void clear() {
    _items.clear();
    notifyListeners();
    // หมายเหตุ: เมธอดนี้ยังไม่ถูกเรียกใช้จากที่ใดในใบงานส่วนที่ 1-4
    // จะถูกนำไปใช้จริงในส่วนที่ 5 (ทำด้วยตนเอง)
  }
}
```

### ขั้นตอนที่ 2.3: ลงทะเบียน Provider ที่ราก main.dart

แก้ไข `lib/main.dart` ให้ครอบทั้งแอปด้วย `ChangeNotifierProvider` ตั้งแต่จุดสูงสุด

```dart
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';
import 'models/favorites_model.dart';
import 'home_page.dart';

void main() {
  runApp(
    // สร้าง FavoritesModel ขึ้นมาหนึ่งตัว แล้วให้ทุก Widget ใต้ MyApp เข้าถึงได้
    ChangeNotifierProvider(
      create: (context) => FavoritesModel(),
      child: const MyApp(),
    ),
  );
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Campus Marketplace',
      debugShowCheckedModeBanner: false, // ปิดริบบิ้น DEBUG มุมขวาบน ไม่ให้บังไอคอนหัวใจใน AppBar
      home: const HomePage(),
    );
  }
}
```

### ขั้นตอนที่ 2.4: เขียน ItemCard ใหม่ให้ดึง Provider เอง (ลบ Prop Drilling ชั้นล่าง)

แทนที่เนื้อหาทั้งหมดใน `lib/widgets/item_card.dart` ด้วยเวอร์ชันนี้ สังเกตว่าพารามิเตอร์ `savedItems` และ `onSave` หายไปจาก constructor ทั้งคู่

```dart
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';
import '../models/item.dart';
import '../models/favorites_model.dart';

class ItemCard extends StatelessWidget {
  final Item item; // เหลือแค่พารามิเตอร์เดียว ไม่ต้องรับ savedItems/onSave อีกต่อไป

  const ItemCard({super.key, required this.item});

  @override
  Widget build(BuildContext context) {
    // .watch ที่นี่เพื่อให้ปุ่มอัปเดตสถานะ "บันทึกแล้ว" ทันทีที่ FavoritesModel เปลี่ยนจากจุดใดก็ตาม
    final favorites = context.watch<FavoritesModel>();
    final alreadySaved = favorites.items.any((i) => i.id == item.id);

    return Card(
      margin: const EdgeInsets.symmetric(horizontal: 12, vertical: 6),
      child: ListTile(
        title: Text(item.title),
        subtitle: Text('฿${item.price.toStringAsFixed(0)}'),
        trailing: ElevatedButton(
          onPressed: alreadySaved
              ? null
              : () {
                  // .read ที่นี่เพราะเป็นคำสั่งครั้งเดียวตอนกด ไม่ต้องการสมัครรับการอัปเดตซ้ำ
                  context.read<FavoritesModel>().add(item);
                  ScaffoldMessenger.of(context).showSnackBar(
                    SnackBar(content: Text('บันทึก ${item.title} ไว้ในรายการโปรดแล้ว')),
                  );
                },
          child: Text(alreadySaved ? '❤️ บันทึกแล้ว' : '🤍 บันทึกเป็นรายการโปรด'),
        ),
      ),
    );
  }
}
```

### ขั้นตอนที่ 2.5: เขียน ItemListSection ใหม่ (ลบ Prop Drilling ชั้นกลาง)

แทนที่เนื้อหาทั้งหมดใน `lib/widgets/item_list_section.dart` — ตอนนี้เหลือแค่พารามิเตอร์เดียวคือ `catalog` เพราะ `ItemCard` แต่ละใบไปดึง `FavoritesModel` เองโดยตรงแล้ว ไม่ต้องพึ่ง Widget แม่ส่งต่อให้อีก

```dart
import 'package:flutter/material.dart';
import '../models/item.dart';
import 'item_card.dart';

class ItemListSection extends StatelessWidget {
  final List<Item> catalog; // เหลือพารามิเตอร์เดียว เพราะ ItemCard ไปดึง FavoritesModel เอง

  const ItemListSection({super.key, required this.catalog});

  @override
  Widget build(BuildContext context) {
    return ListView.builder(
      shrinkWrap: true,
      physics: const NeverScrollableScrollPhysics(),
      itemCount: catalog.length,
      // สังเกตว่าตอนนี้ ItemListSection ไม่ต้องรู้จัก FavoritesModel เลยด้วยซ้ำ
      itemBuilder: (context, index) => ItemCard(item: catalog[index]),
    );
  }
}
```

### ขั้นตอนที่ 2.6: สร้างหน้า FavoritesPage แยก Route

สร้างไฟล์ `lib/favorites_page.dart` เป็นหน้าจอใหม่ที่จะถูก push แยกออกไปจาก `HomePage` ให้แสดงรายการสินค้าที่บันทึกไว้พร้อมมูลค่ารวม โดยดึงข้อมูลจาก `context.watch<FavoritesModel>()` **ห้ามส่งข้อมูลรายการโปรดผ่าน constructor ของหน้านี้โดยเด็ดขาด** — นี่คือจุดที่พิสูจน์ว่า Provider แก้ปัญหาข้าม Route ได้จริง

⚠️ **ต้องสร้างไฟล์นี้ก่อนขั้นตอนที่ 2.7 เสมอ** เพราะ `home_page.dart` เวอร์ชันถัดไปจะ `import 'favorites_page.dart'` และเรียกใช้ `FavoritesPage()` ตรง ๆ ถ้าสร้าง `home_page.dart` ก่อนไฟล์นี้จะยังไม่มีอยู่จริง แอปจะไม่รันได้เลย

```dart
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';
import 'models/favorites_model.dart';

class FavoritesPage extends StatelessWidget {
  const FavoritesPage({super.key});

  @override
  Widget build(BuildContext context) {
    // .watch เพราะหน้านี้ต้อง rebuild ทุกครั้งที่รายการโปรดเปลี่ยน (เช่น กดลบจากหน้านี้เอง)
    final favorites = context.watch<FavoritesModel>();

    return Scaffold(
      appBar: AppBar(title: const Text('รายการโปรดของฉัน')),
      body: favorites.items.isEmpty
          ? const Center(child: Text('ยังไม่มีสินค้าที่บันทึกไว้'))
          : ListView.builder(
              itemCount: favorites.items.length,
              itemBuilder: (context, index) {
                final item = favorites.items[index];
                return ListTile(
                  title: Text(item.title),
                  subtitle: Text('฿${item.price.toStringAsFixed(0)}'),
                  trailing: IconButton(
                    icon: const Icon(Icons.delete_outline),
                    // .read เพราะเป็นการกดปุ่มครั้งเดียว ไม่ใช่การอ่านค่าต่อเนื่องแบบ .watch
                    onPressed: () => context.read<FavoritesModel>().remove(item),
                  ),
                );
              },
            ),
      bottomNavigationBar: Padding(
        padding: const EdgeInsets.all(12),
        child: Text('มูลค่ารวม: ฿${favorites.totalValue.toStringAsFixed(0)}'),
      ),
    );
  }
}
```

### ขั้นตอนที่ 2.7: เขียน HomePage ใหม่ (จาก StatefulWidget กลับมาเป็น StatelessWidget)

แทนที่เนื้อหาทั้งหมดใน `lib/home_page.dart` — จุดที่น่าสังเกตที่สุดคือ `HomePage` ไม่ต้องเก็บ State อะไรไว้เองอีกแล้ว จึงเปลี่ยนกลับจาก `StatefulWidget` เป็น `StatelessWidget` ธรรมดาได้ (ไฟล์นี้ import `favorites_page.dart` ที่สร้างไว้ในขั้นตอนที่ 2.6 ดังนั้นต้องทำขั้นตอนที่ 2.6 ให้เสร็จก่อนเสมอ)

```dart
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';
import 'models/item.dart';
import 'models/favorites_model.dart';
import 'widgets/item_list_section.dart';
import 'favorites_page.dart';

class HomePage extends StatelessWidget {
  // เปลี่ยนจาก StatefulWidget เป็น StatelessWidget ได้เลย เพราะไม่ต้องเก็บ State ใด ๆ ไว้เองอีกแล้ว
  const HomePage({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Campus Marketplace'),
        actions: [
          IconButton(
            icon: Row(
              mainAxisSize: MainAxisSize.min,
              children: [
                const Icon(Icons.favorite),
                // .watch ทำให้ตัวเลขนี้อัปเดตเองทุกครั้งที่ FavoritesModel เปลี่ยน ไม่ว่าจะเปลี่ยนจากจุดไหน
                Text(' ${context.watch<FavoritesModel>().itemCount}'),
              ],
            ),
            onPressed: () => Navigator.push(
              context,
              MaterialPageRoute(builder: (_) => const FavoritesPage()),
            ),
          ),
        ],
      ),
      body: ItemListSection(catalog: catalog),
    );
  }
}
```

> ✅ **Checkpoint 2.1** รันแอปใหม่ ทดสอบกดบันทึกสินค้าจากหลายจุด แล้วตรวจว่าตัวเลขที่ AppBar อัปเดตถูกต้องทุกครั้ง โดยที่ไฟล์ `item_list_section.dart` และ `item_card.dart` **ไม่มีพารามิเตอร์ savedItems หรือ onSave หลงเหลือใน constructor แล้ว**

<img width="954" height="909" alt="Screenshot 2026-08-14 104813" src="https://github.com/user-attachments/assets/42c5751f-b138-49f6-8ad4-87aef1a4bc47" />


<img width="818" height="266" alt="Screenshot 2026-08-14 104908" src="https://github.com/user-attachments/assets/992c89eb-bc7b-48d4-9dd5-0bfe8b5dd933" />



<img width="803" height="259" alt="Screenshot 2026-08-14 104918" src="https://github.com/user-attachments/assets/da78f45c-4313-4c79-a4c1-cf30a1521e74" />


> ✅ **Checkpoint 2.2** ทดสอบว่าเมื่อบันทึกสินค้าจากหน้า Home แล้วกดไปหน้า Favorites ตัวเลขและรายการสินค้าตรงกันทันที ลองกดปุ่มถังขยะลบสินค้าออกจากหน้า Favorites แล้วย้อนกลับไปหน้า Home ดูว่าปุ่มของสินค้านั้นกลับมากดซ้ำได้อีกครั้ง ถ่ายภาพหน้าจอทั้งสองหน้าเทียบกันแนบส่ง

<img width="955" height="904" alt="image" src="https://github.com/user-attachments/assets/3e0386fd-581b-4892-a764-6f4912cc47f8" />


<img width="960" height="911" alt="image" src="https://github.com/user-attachments/assets/3472e501-24c5-46b6-8bee-2ba586750712" />

<img width="954" height="904" alt="image" src="https://github.com/user-attachments/assets/75bd6048-4383-4b7f-883e-f7aa7593862e" />

<img width="954" height="901" alt="image" src="https://github.com/user-attachments/assets/99ba64a5-3d32-4096-9deb-9582d6300449" />

---

## ส่วนที่ 3: ใช้ AI ช่วยเลือกแนวทาง State Management

### ขั้นตอนที่ 3.1

เปิด Google AI Studio (https://aistudio.google.com) แล้วสร้าง Prompt นี้กับ Gemini

```
ฉันกำลังพัฒนาแอป Flutter ตลาดนัดออนไลน์ (Campus Marketplace) ที่มีฟีเจอร์ต่อไปนี้:
1. Dark Mode / Light Mode ที่ต้องส่งผลต่อทุกหน้าจอในแอป
2. ตัวนับ "มีคนถูกใจแล้วกี่คน" ของประกาศขายสินค้า ที่ต้องซิงค์ระหว่างหน้ารายการประกาศกับหน้ารายละเอียดสินค้า
3. Animation กระพริบของไอคอนหัวใจตอนกดถูกใจ ที่ใช้เฉพาะใน widget เดียว

ช่วยวิเคราะห์ว่าแต่ละฟีเจอร์ควรใช้ setState, Provider หรือ Riverpod
และอธิบายเหตุผลของแต่ละข้อสั้น ๆ
```

บันทึกคำตอบที่ได้จาก Gemini 

```text

สำหรับการพัฒนาแอป Flutter ในปัจจุบัน ผมแนะนำให้ใช้ Riverpod เป็นหลักสำหรับ State ที่มีการแชร์ข้อมูลกัน และใช้ setState สำหรับ UI Logic เล็กๆ น้อยๆ ภายใน Widget ครับ
นี่คือบทวิเคราะห์ตามโจทย์ของคุณ:
1. Dark Mode / Light Mode (Global Theme)
ควรใช้: Riverpod (หรือ Provider)
เหตุผล:
Global State: ธีมเป็นสถานะที่ส่งผลกระทบต่อ ทั้งแอป (App-wide) ทุกหน้าจอต้องเข้าถึงค่านี้ได้
Efficiency: การใช้ Riverpod (เช่น StateProvider<ThemeMode>) ช่วยให้คุณเปลี่ยนธีมได้จากหน้า Setting และทุก Widget ที่ฟังค่าอยู่จะ Rebuild พร้อมกันทันทีโดยไม่ต้องส่งค่าผ่าน Constructor (Prop Drilling)
2. ตัวนับ "จำนวนคนถูกใจ" (Synced between screens)
ควรใช้: Riverpod (หรือ Provider)
เหตุผล:
Shared State: ข้อมูลนี้ถูกใช้ใน 2 หน้าจอ (List View และ Detail View) หากคุณกดถูกใจในหน้า Details แล้วกดย้อนกลับมาหน้า List ข้อมูลต้องเป็นตัวเลขเดียวกัน
Single Source of Truth: การใช้ Riverpod (เช่น StateNotifierProvider หรือ AsyncNotifierProvider) จะช่วยเก็บสถานะของสินค้าไว้ที่ส่วนกลาง เมื่อมีการแก้ไขข้อมูลที่จุดหนึ่ง อีกจุดที่เรียกใช้ข้อมูลชุดเดียวกันจะอัปเดตอัตโนมัติ
3. Animation กระพริบของไอคอนหัวใจ (Local Animation)
ควรใช้: setState
เหตุผล:
Local State: แอนิเมชันนี้เกิดขึ้นเฉพาะภายใน Widget ตัวเดียว (เช่น LikeButtonWidget) และไม่มีหน้าจออื่นจำเป็นต้องรู้ว่าไอคอนกำลังกระพริบอยู่หรือไม่
Performance & Simplicity: การใช้ AnimationController ร่วมกับ setState (หรือ AnimatedBuilder) ภายในสโคปเล็กๆ เป็นวิธีที่เหมาะสมที่สุด ไม่ทำให้แอปซับซ้อนเกินความจำเป็น และลดภาระของ Global State Management
สรุปคำแนะนำ:
ฟีเจอร์	ตัวเลือกที่แนะนำ	เหตุผลหลัก
Theme	Riverpod	เป็น Global State ที่ต้องใช้ร่วมกันทั้งแอป
Like Counter	Riverpod	ต้องซิงค์ข้อมูลระหว่างหน้าจอ (Cross-screen sync)
Like Animation	setState	เป็น UI Logic เฉพาะตัว Widget ไม่ต้องแชร์กับใคร
Pro Tip: หากโปรเจกต์นี้ต้องการสเกลในอนาคต การเริ่มด้วย Riverpod จะช่วยให้จัดการ Error Handling และการดึงข้อมูลจาก API (Async Data) ได้ง่ายกว่า Provider ครับ

```


### ขั้นตอนที่ 3.2: ประเมินคำตอบของ AI

เปรียบเทียบคำตอบของ Gemini กับกรอบการตัดสินใจในบทหนังสือเรียนหัวข้อ 5.7 แล้วตอบคำถามต่อไปนี้

- Gemini แนะนำตรงกับกรอบการตัดสินใจในบทเรียนหรือไม่ มีจุดใดที่ต่างกัน
  
```text
คำตอบของ AI ตรงกับกรอบการตัดสินใจในบทเรียนเรื่องการแบ่งแยกขอบเขตการใช้งาน คือแนะนำให้ใช้ `setState` กับข้อมูล Widget เดียว (Ephemeral State เช่น แอนิเมชัน)
และให้ใช้เครื่องมืออื่นที่จัดการข้อมูลส่วนกลางสำหรับข้อมูลข้ามหน้าจอ (App State เช่น ธีม และ ตัวนับถูกใจ)
แต่จุดที่แตกต่างกัน คือ ลำดับการเลือกใช้เครื่องมือ
- ตามหลักในบทเรียนข้อ 5.7 แนะนำว่าข้อมูลที่ใช้ข้ามหลายหน้าจอ ให้ยกระดับไปใช้ Provider ก่อน แล้วย้ายไป Riverpod ตอนที่ต้องการ Type Safety สูงขึ้น
- แต่ Gemini แนะนำให้ใช้ Riverpod เป็นตัวเลือกหลัก (Default) สำหรับฟีเจอร์ที่ 1 และ 2 โดยให้ Provider เป็นแค่ตัวเลือกสำรอง 

```
- หากคำตอบของ Gemini ดูสมเหตุสมผลแต่ยังไม่ครบถ้วน (เช่น ไม่ได้พูดถึงขอบเขตของ Widget) ให้ลองถามคำถามต่อเพื่อขอเหตุผลเพิ่มเติม แล้วบันทึกบทสนทนาไว้ด้วย

เราถาม AI
```
ตามกรอบการตัดสินใจของบทเรียน แนะนำว่าถ้าต้องแชร์ข้อมูลหลายหน้าจอ ให้เริ่มจาก Provider ก่อน แล้วถ้าต้องการ Type Safety หรือการเทสต์ที่ง่ายขึ้น ค่อยใช้
Riverpod แต่ทำไมคุณถึงแนะนำให้ใช้ Riverpod เป็นตัวเลือกหลักล่ะ? การเริ่มด้วย Provider สำหรับ 2 ฟีเจอร์นี้มีข้อเสียอะไรไหมในมุมมองของคุณ?

```
AI ตอบ
```
เป็นคำถามที่ดีมากครับ และผมเข้าใจกรอบการเรียนรู้ที่คุณกล่าวถึง (ซึ่งมักจะสอนให้เริ่มจากพื้นฐานไปหาความซับซ้อน) แต่ในมุมมองของการพัฒนาแอปจริงในปัจจุบัน นี่คือเหตุผลที่ผมข้าม Provider และแนะนำ Riverpod เป็นตัวเลือกหลัก รวมถึงข้อเสียบางประการของ Provider ที่คุณอาจต้องเจอครับ:
1. ปัญหาเรื่อง "Runtime Exceptions" vs "Compile-time Safety"
Provider: ทำงานอิงกับ Widget Tree ถ้าคุณเรียกใช้ Provider.of<T>(context) ในหน้าที่ไม่ได้ห่อ (Wrap) ด้วย Provider ตัวนั้นไว้ แอปจะ Crash ทันทีในขณะรัน (Runtime) ซึ่งบางครั้งหาจุดผิดยากในแอปขนาดใหญ่
Riverpod: ตัวแปร Provider จะถูกประกาศเป็น Global (แต่เก็บ State แยกกัน) ทำให้คุณ ไม่มีทางเจอ ProviderNotFoundException เพราะมันไม่ได้ขึ้นอยู่กับลำดับชั้นของ Widget Tree ครับ ความปลอดภัยนี้ช่วยลด Bug ได้มหาศาล
2. ความยุ่งยากเมื่อต้องใช้ BuildContext
Provider: จำเป็นต้องมี BuildContext เสมอในการเข้าถึงข้อมูล ซึ่งในแอปตลาดนัด (Marketplace) เมื่อแอปเริ่มซับซ้อน คุณอาจจะอยากจัดการ Logic ในชั้น Service หรือ Repository (เช่น หลังจากส่ง API ตรวจสอบการถูกใจเสร็จ ให้ไปสั่งอัปเดตข้อมูลหน้าอื่น) ซึ่งในชั้นเหล่านั้น ไม่มี BuildContext ทำให้ Provider ใช้งานยากขึ้น
Riverpod: เข้าถึงได้ผ่านวัตถุ ref ซึ่งไม่ต้องพึ่งพา context ทำให้คุณแยก Logic ออกจาก UI ได้สะอาดกว่ามาก
3. การจัดการ State ที่ซับซ้อน (เช่น "มีคนถูกใจกี่คน")
ข้อเสียของ Provider: หากข้อมูล "จำนวนคนถูกใจ" ต้องโหลดมาจาก API (Asynchronous) การจัดการ Loading, Error และ Data ใน Provider ต้องเขียน Code เยอะ (Boilerplate) หรือต้องสร้าง Class แยกเพื่อเก็บสถานะเหล่านี้
Riverpod: มี AsyncNotifier หรือ FutureProvider ที่จัดการสถานะ Loading/Error ให้ในตัว (มีเมธอด .when()) ซึ่งเหมาะมากกับแอป Marketplace ที่ต้องดึงข้อมูลจาก Server ตลอดเวลา
4. การจัดการหน่วยความจำ (Memory Management)
Provider: การทำลาย State ทิ้งเมื่อไม่ใช้งาน (Dispose) ทำได้ยากกว่าเล็กน้อย ถ้าเราประกาศไว้ที่หน้าแรกสุด (Root) ของแอป มันจะอยู่ค้างใน Memory ตลอดไป
Riverpod: มีฟีเจอร์ .autoDispose แค่เติมคำนี้ลงไป เมื่อหน้านั้นๆ ไม่ถูกใช้งาน Riverpod จะคืนค่า Memory ให้ระบบโดยอัตโนมัติ เหมาะกับหน้ารายละเอียดสินค้า (Detail Page) ที่เราอยากให้ล้างข้อมูลทิ้งเมื่อผู้ใช้ปิดหน้านั้นไป
ถ้าคุณจะเริ่มด้วย Provider มีข้อเสียร้ายแรงไหม?
คำตอบคือ: ไม่ร้ายแรงครับ ถ้าคุณกำลังเรียนรู้ การเริ่มด้วย Provider มีข้อดีคือ:
เข้าใจพื้นฐานของ Flutter ดีขึ้น: เพราะ Provider ทำงานล้อไปกับระบบ InheritedWidget ของ Flutter
เอกสารเยอะ: เนื่องจากมีมานาน คำถามใน Stack Overflow จะมีคำตอบสำหรับ Provider เยอะมาก
แต่เหตุผลที่ผมแนะนำ Riverpod เลย เพราะในโปรเจกต์ Marketplace คุณจะเจอเคส "อัปเดตข้อมูลข้ามหน้า" และ "จัดการข้อมูลจาก API" เร็วมาก ซึ่ง Riverpod จะช่วยให้คุณเขียนโค้ดได้สั้นลง ปลอดภัยขึ้น และไม่ต้องมานั่งแก้ปัญหา "หา Provider ไม่เจอ" ที่มักจะเป็นปัญหาอันดับหนึ่งของมือใหม่ที่ใช้ Provider ครับ
สรุป:
ถ้าต้องการ เรียนรู้รากฐาน -> เริ่มที่ Provider
ถ้าต้องการ เขียนแอปใช้งานจริง/ระยะยาว -> ไปที่ Riverpod ได้เลยครับ ไม่ต้องเสียเวลาเปลี่ยนทีหลัง :)

```


⚠️ **ข้อควรระวัง**: AI เป็นเครื่องมือช่วยคิด ไม่ใช่คำตอบสุดท้าย ผู้เรียนต้องอธิบายเหตุผลของการเลือกใช้เครื่องมือได้ด้วยตัวเองเสมอ ตามหลักการใช้ AI ในการพัฒนาซอฟต์แวร์ของวิชานี้

---

## ส่วนที่ 4 (เพิ่มเติม/ทดลอง): แปลง FavoritesModel เป็น Riverpod

ส่วนนี้เป็นแบบฝึกหัดเสริมเพื่อให้เห็นความแตกต่างของโค้ดจริงระหว่าง Provider และ Riverpod **ไม่บังคับเปลี่ยนโปรเจกต์หลัก** `campus_marketplace` แต่ให้สร้าง**โปรเจกต์ทดลองใหม่แยกต่างหากทั้งหมด** เพื่อเทียบโค้ดแบบเดียวกันที่เขียนด้วย Provider (ส่วนที่ 2) กับ Riverpod แบบเคียงข้างกัน

**ภาพรวมก่อนเริ่ม**: โปรเจกต์ทดลองนี้จะมีแค่ 3 ไฟล์เท่านั้น ต่างจากโปรเจกต์หลักที่แยกเป็นหลายไฟล์/โฟลเดอร์ (ไม่มี `ItemCard`, `ItemListSection`, `FavoritesPage` แยก) เพราะจุดประสงค์คือเทียบไวยากรณ์ให้เห็นชัด ไม่ใช่สร้างแอปสมบูรณ์อีกรอบ

```
campus_marketplace_riverpod_trial/
└── lib/
    ├── item.dart               ← โมเดลข้อมูล เหมือนขั้นตอนที่ 1.1
    ├── favorites_notifier.dart ← เทียบเท่า FavoritesModel แต่เขียนแบบ Riverpod
    └── main.dart                ← รวม MyApp + HomePage ไว้ในไฟล์เดียว เพื่อความกระชับ
```

### ขั้นตอนที่ 4.1: สร้างโปรเจกต์ใหม่

เปิด Terminal แล้วรันคำสั่งนี้ **นอกโฟลเดอร์ `campus_marketplace` เดิม** (อย่าสร้างโปรเจกต์ซ้อนในโปรเจกต์)

```bash
flutter create campus_marketplace_riverpod_trial
```

`cd` เข้าไปในโฟลเดอร์ที่เพิ่งสร้าง แล้วเปิดด้วย VS Code ผ่านเมนู **File → Open Folder...** (เลือกโฟลเดอร์ `campus_marketplace_riverpod_trial` ที่มี `pubspec.yaml` อยู่ข้างใน) จากนั้นเปิดไฟล์ `lib/main.dart` ที่ Flutter สร้างให้อัตโนมัติ แล้ว **ลบโค้ด Counter Demo เริ่มต้นทั้งหมดทิ้งให้เหลือไฟล์ว่าง** เราจะเขียนเนื้อหาใหม่ทั้งไฟล์ในขั้นตอนที่ 4.5

### ขั้นตอนที่ 4.2: ติดตั้งแพ็กเกจ

เปิดไฟล์ `pubspec.yaml` (อยู่ที่ root ของโปรเจกต์ทดลองนี้ คนละไฟล์กับ `pubspec.yaml` ของ `campus_marketplace`) แล้วเพิ่มบรรทัดนี้ในส่วน `dependencies:`

```yaml
dependencies:
  flutter:
    sdk: flutter
  flutter_riverpod: ^2.5.1
```

รันคำสั่งในเทอร์มินัล (ต้องอยู่ในโฟลเดอร์ `campus_marketplace_riverpod_trial` ตามหลักการเดียวกับที่อธิบายไว้ในหัวข้อ Troubleshooting เรื่อง `pubspec.yaml`)

```bash
flutter pub get
```

### ขั้นตอนที่ 4.3: สร้าง Item Model

เนื่องจากนี่คือโปรเจกต์ Flutter ใหม่แยกต่างหาก จึงยังไม่มีคลาส `Item` หรือ `catalog` อยู่เลย (ไม่ได้สืบทอดไฟล์จากโปรเจกต์หลักให้อัตโนมัติ) ให้สร้างไฟล์ใหม่ชื่อ `lib/item.dart` แล้วใส่เนื้อหาเดียวกับที่สร้างไว้ในขั้นตอนที่ 1.1 ของโปรเจกต์หลัก (วางไว้ที่ `lib/item.dart` ตรง ๆ ไม่ต้องมีโฟลเดอร์ `models/` ซ้อนอีกชั้น เพราะโปรเจกต์ทดลองนี้ตั้งใจให้มีโครงสร้างเรียบง่ายที่สุด)

```dart
class Item {
  final String id;
  final String title;
  final double price;

  const Item({required this.id, required this.title, required this.price});
}

final catalog = <Item>[
  const Item(id: 'i1', title: 'หนังสือ Calculus มือสอง', price: 150),
  const Item(id: 'i2', title: 'หูฟังไร้สาย (สภาพดี 90%)', price: 450),
  const Item(id: 'i3', title: 'โคมไฟตั้งโต๊ะหอพัก', price: 120),
];
```

### ขั้นตอนที่ 4.4: สร้าง FavoritesNotifier (เทียบเท่า FavoritesModel)

สร้างไฟล์ใหม่ชื่อ `lib/favorites_notifier.dart` — คลาสนี้ทำหน้าที่เดียวกับ `FavoritesModel` ในส่วนที่ 2 ทุกเมธอด (add, remove, totalValue) เพียงแต่เปลี่ยนวิธีเก็บ State จากตัวแปร mutable ภายในคลาสมาเป็นการ "แทนที่ state ก้อนใหม่" ทั้งหมดทุกครั้งที่แก้ไข ตารางนี้ช่วยให้เห็นว่ากำลังแปลงอะไรเป็นอะไร

| ในโปรเจกต์หลัก (Provider) | ในโปรเจกต์ทดลองนี้ (Riverpod) |
|---|---|
| `class FavoritesModel extends ChangeNotifier` | `class FavoritesNotifier extends StateNotifier<List<Item>>` |
| `final List<Item> _items = []` + `notifyListeners()` | `state = [...state, item]` (แทนที่ก้อนใหม่ทั้งหมด) |
| ลงทะเบียนด้วย `ChangeNotifierProvider` | ลงทะเบียนด้วย `StateNotifierProvider` |

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'item.dart';

class FavoritesNotifier extends StateNotifier<List<Item>> {
  FavoritesNotifier() : super([]); // ค่าเริ่มต้นคือลิสต์ว่าง เทียบเท่า _items = [] ใน ChangeNotifier

  // ใช้ spread operator [...state, item] สร้างลิสต์ใหม่ทั้งก้อน แทนการ mutate ลิสต์เดิม
  void add(Item item) => state = [...state, item];

  // เช่นเดียวกัน ใช้ .where() สร้างลิสต์ใหม่ที่ไม่มีไอเทมนี้อยู่ แทนการ remove ตรง ๆ
  void remove(Item item) => state = state.where((i) => i.id != item.id).toList();

  double get totalValue => state.fold(0, (sum, i) => sum + i.price);
}

// ประกาศ Provider เป็นตัวแปร global เทียบเท่ากับการลงทะเบียน ChangeNotifierProvider ใน main.dart
final favoritesProvider = StateNotifierProvider<FavoritesNotifier, List<Item>>(
  (ref) => FavoritesNotifier(),
);
```

### ขั้นตอนที่ 4.5: เขียน main.dart ใหม่ทั้งไฟล์

ต่างจากโปรเจกต์หลักที่แยก `HomePage` ไว้คนละไฟล์ ในโปรเจกต์ทดลองนี้ให้รวม `MyApp` และ `HomePage` ไว้ใน `main.dart` ไฟล์เดียวเพื่อความกระชับ เปิดไฟล์ `lib/main.dart` ที่ลบโค้ดเดิมทิ้งไว้แล้วตั้งแต่ขั้นตอนที่ 4.1 แล้วพิมพ์โค้ดนี้ลงไปทั้งหมด

```dart
import 'package:flutter/material.dart';
import 'package:flutter_riverpod/flutter_riverpod.dart';
import 'item.dart';
import 'favorites_notifier.dart';

void main() {
  // ครอบแอปทั้งหมดด้วย ProviderScope เพียงครั้งเดียวที่จุดเริ่มต้น เทียบเท่า ChangeNotifierProvider
  runApp(const ProviderScope(child: MyApp()));
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});
  @override
  Widget build(BuildContext context) => const MaterialApp(
        debugShowCheckedModeBanner: false, // ปิดริบบิ้น DEBUG มุมขวาบน ไม่ให้บังไอคอนหัวใจใน AppBar
        home: HomePage(),
      );
}

// ใช้ ConsumerWidget แทน StatelessWidget เพื่อรับพารามิเตอร์ "ref" เข้ามาใน build()
class HomePage extends ConsumerWidget {
  const HomePage({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    // ref.watch อ่านค่าปัจจุบันและสมัครรับการอัปเดตอัตโนมัติ เทียบเท่า context.watch
    final savedItems = ref.watch(favoritesProvider);

    return Scaffold(
      appBar: AppBar(title: Text('❤️ ${savedItems.length}')),
      body: ListView(
        children: catalog.map((item) => ListTile(
          title: Text(item.title),
          trailing: ElevatedButton(
            // ref.read(...notifier) ใช้เรียกแก้ไขค่า เทียบเท่า context.read
            onPressed: () => ref.read(favoritesProvider.notifier).add(item),
            child: const Text('บันทึก'),
          ),
        )).toList(),
      ),
    );
  }
}
```

> ✅ **Checkpoint 4.1** รันแอปด้วย `flutter run` (หรือกด F5 ใน VS Code) แล้วทดสอบกดปุ่ม "บันทึก" ที่สินค้าชิ้นใดก็ได้ ตรวจว่าตัวเลข ❤️ ที่ AppBar เพิ่มขึ้นถูกต้อง ถ่ายภาพหน้าจอแนบส่ง

<img width="1296" height="872" alt="image" src="https://github.com/user-attachments/assets/32b4572e-ae07-42be-b9c1-54301d149def" />


> ✅ **Checkpoint 4.2** เขียนตารางเปรียบเทียบสั้น ๆ ว่าตอนแปลงจาก Provider เป็น Riverpod ต้องเปลี่ยนอะไรบ้าง (เช่น `ChangeNotifier` → `StateNotifier`, `StatelessWidget` → `ConsumerWidget`, `context.watch` → `ref.watch`) อย่างน้อย 4 คู่เทียบ


| สิ่งที่ต้องเปลี่ยน |  Provider |  Riverpod |
| :--- | :--- | :--- |
| **1. คลาสที่ใช้จัดการ State** | `extends ChangeNotifier` | `extends StateNotifier<List<Item>>` |
| **2. เริ่มต้นแอป** | ครอบด้วย `ChangeNotifierProvider` | ครอบด้วย `ProviderScope` |
| **3. ชนิด Widget ที่ใช้อ่านค่า** | `StatelessWidget` | `ConsumerWidget` |
| **4. การอัปเดตข้อมูล** | แก้ไข List เดิม แล้วสั่ง `notifyListeners()` | แทนที่ด้วย State ก้อนใหม่ทั้งหมด `state = [...]` |

---

## ส่วนที่ 5 (ทำด้วยตนเอง): ออกแบบฟีเจอร์เพิ่มด้วยตัวเอง

ส่วนนี้**ไม่มีโค้ดต้นแบบให้ทั้งหมด**เหมือนส่วนก่อนหน้า เพราะจุดประสงค์คือให้ผู้เรียนนำหลักการ Ephemeral State vs App State และการใช้ Provider ที่เรียนมาทั้งบท ไปประยุกต์ออกแบบและเขียนโค้ดด้วยตนเอง จำลองสถานการณ์ทำงานจริงที่ไม่มีใบสั่งงานบอกทุกขั้นตอน

### โจทย์ที่ 1: ช่องค้นหาสินค้า (Search Box)

เพิ่ม `TextField` ที่หน้า Home สำหรับพิมพ์คำค้นหา แล้วกรองรายการที่ส่งให้ `ItemListSection` ให้เหลือเฉพาะสินค้าที่ `title` มีคำค้นหาอยู่ (ไม่สนตัวพิมพ์เล็ก-ใหญ่)

**ข้อกำหนด**

- ต้องตัดสินใจเองว่าค่าคำค้นหาควรเป็น Ephemeral State หรือ App State พร้อมให้เหตุผลสั้น ๆ ไว้ใน `notes.md` (คำใบ้: ตอนนี้มีหน้าจออื่นในแอปที่ต้องรู้ค่าคำค้นหานี้ด้วยหรือไม่)
- ถ้าตัดสินใจว่าเป็น Ephemeral State ห้ามใช้ Provider สำหรับฟีเจอร์นี้ ให้ฝึกเลือกใช้เครื่องมือที่เบาที่สุดที่เพียงพอ (`setState` ธรรมดา)

### โจทย์ที่ 2: ปุ่ม "ล้างรายการโปรดทั้งหมด"

สังเกตว่า `FavoritesModel` มีเมธอด `clear()` เตรียมไว้ให้แล้วตั้งแต่ขั้นตอนที่ 2.2 แต่ยังไม่เคยถูกเรียกใช้งานจากที่ใดเลย ให้เพิ่มปุ่มในหน้า `FavoritesPage` ที่เรียกใช้เมธอดนี้ พร้อมแสดง Dialog ยืนยันก่อนล้างข้อมูลจริง (ใช้ `showDialog` + `AlertDialog`)

**ข้อกำหนด**

- ต้องใช้ `context.read` หรือ `context.watch` ให้ถูกต้องตามหลักการ และอธิบายเหตุผลการเลือกไว้ใน `notes.md`
- ปุ่มต้องแสดงเฉพาะเมื่อมีรายการโปรดอย่างน้อย 1 รายการเท่านั้น (ถ้ารายการว่างอยู่แล้วไม่ต้องแสดงปุ่มนี้)

### โจทย์ที่ 3 (ท้าทายเพิ่ม ไม่บังคับ)

ทำโจทย์ที่ 1 และ 2 ซ้ำอีกครั้งในโปรเจกต์ทดลอง Riverpod (ส่วนที่ 4) เพื่อฝึกโยกความสามารถเดียวกันข้ามเครื่องมือ State Management สองแบบ

> ✅ **Checkpoint 5.1** ถ่ายภาพหน้าจอฟีเจอร์ค้นหาที่กรองสินค้าได้ถูกต้อง และภาพ Dialog ยืนยันการล้างรายการโปรด เขียนอธิบายเหตุผลการเลือกชนิด State ของทั้งสองฟีเจอร์
```text
โจทย์ที่ 1: ช่องค้นหาสินค้า
- เลือกใช้ Ephemeral State (ผ่าน `setState`) เพราะ คำค้นหาเป็นข้อมูลที่ใช้กรองรายการสินค้าเฉพาะภายในหน้า `HomePage` ไม่มีหน้าจออื่นในแอปที่จำเป็นต้องรู้ข้อมูลนี้ 

โจทย์ที่ 2: ปุ่มล้างรายการโปรดทั้งหมด
- การแสดงปุ่ม: ใช้ `context.watch` เพราะให้ AppBar ตรวจสอบสถานะอยู่ตลอดเวลา และซ่อนปุ่มโดยอัตโนมัติเมื่อรายการโปรดว่างเปล่า
- การกดปุ่มเพื่อลบ: ใช้ `context.read` ภายใน `onPressed` ของปุ่มยืนยันใน Dialog เพราะส่งคำสั่งลบข้อมูลเพียงครั้งเดียว (Event)

```


<img width="1291" height="874" alt="image" src="https://github.com/user-attachments/assets/7935d105-9e9d-4317-a752-1203622d8db5" />



<img width="1305" height="877" alt="image" src="https://github.com/user-attachments/assets/269a84cb-4564-48bd-93bc-6f0ad629f48d" />



<img width="1296" height="879" alt="image" src="https://github.com/user-attachments/assets/12657f91-1d2b-47eb-bb32-042a0f70e8a1" />



<img width="1298" height="873" alt="image" src="https://github.com/user-attachments/assets/80237716-4897-437b-ae5f-216d598a9549" />







