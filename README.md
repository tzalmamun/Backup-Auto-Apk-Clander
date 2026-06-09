# 🌌 Sky Backup - Personal Cloud Vault (Secret Sync)

Sky Backup একটি সিকিউর পার্সোনাল ব্যাকআপ সিস্টেম। এই অ্যাপটি বাইরে থেকে দেখতে একটি সাধারণ ডিজিটাল ঘড়ি (Digital Clock) মনে হলেও, স্ক্রিনে ৩ বার দ্রুত ট্যাপ করলে এটি একটি সিক্রেট ব্যাকআপ ড্যাশবোর্ড ওপেন করে। এটি স্বয়ংক্রিয়ভাবে আপনার ফোনের নির্দিষ্ট ফোল্ডার থেকে ফটো এবং স্ক্রিনশট আপনার নিজস্ব হোস্টিং সার্ভারে ব্যাকআপ রাখে।

## 🚀 মূল ফিচারসমূহ:
- **Stealth Mode:** বাইরে থেকে ঘড়ি মনে হয়, ৩-ট্যাপে আসল ড্যাশবোর্ড।
- **Security Lock:** অ্যাপে নিজস্ব পাসওয়ার্ড এবং মাস্টার পাসওয়ার্ড (132011) সিস্টেম।
- **Duplicate Filter:** SHA-256 হ্যাশ ব্যবহার করে ডুপ্লিকেট ফাইল আপলোড বন্ধ রাখা হয়েছে।
- **Folder Selection:** ফোনের যেকোনো ফোল্ডার (Camera, Screenshots, WhatsApp) কাস্টমভাবে সিলেক্ট করার সুবিধা।
- **Live Status:** অ্যাপ ড্যাশবোর্ডে সার্ভারের সাথে কানেকশন লাইভ ইন্ডিকেটর (LED)।
- **Cloud Manager:** সার্ভারের ফাইল সরাসরি অ্যাপ থেকে দেখা এবং ডিলিট করার সুবিধা।
- **Background Sync:** ফোন রিস্টার্ট হলেও ব্যাকগ্রাউন্ডে অটো-ব্যাকআপ সচল থাকে।

---

## 🛠 সার্ভার সেটআপ গাইড (Server Setup Guide)

আপনার নিজস্ব হোস্টিংয়ে এই সিস্টেমটি চালু করতে নিচের ধাপগুলো অনুসরণ করুন:

### ১. হোস্টিংয়ে ফোল্ডার তৈরি
আপনার cPanel এর `public_html` ফোল্ডারে যান এবং `sky` নামে একটি নতুন ফোল্ডার তৈরি করুন। সেই `sky` ফোল্ডারের ভেতর `uploads` নামে আরেকটি ফোল্ডার তৈরি করুন।

### ২. API ফাইল আপলোড
নিচে দেওয়া `api.php` কোডটি কপি করুন এবং আপনার হোস্টিংয়ের `sky/` ফোল্ডারের ভেতর `api.php` নামে সেভ করুন।

### ৩. ফাইল পারমিশন
নিশ্চিত করুন যে `uploads` ফোল্ডারটির পারমিশন `755` অথবা `777` দেওয়া আছে যাতে অ্যাপ ফাইল রাইট করতে পারে।

---

## 📄 api.php (সম্পূর্ণ সার্ভার কোড)
এটি কপি করে আপনার সার্ভারের `api.php` ফাইলে পেস্ট করুন:

```php
<?php
/**
 * Sky Backup Server Side API
 */
header('Content-Type: application/json');
$upload_dir = "uploads/";

if (!file_exists($upload_dir)) {
    mkdir($upload_dir, 0777, true);
}

$action = $_POST['action'] ?? $_GET['action'] ?? '';

switch ($action) {
    case 'upload':
        $file_hash = $_POST['hash'] ?? '';
        $extension = pathinfo($_FILES["file"]["name"], PATHINFO_EXTENSION);
        $target_file = $upload_dir . $file_hash . "." . $extension;

        if (file_exists($target_file)) {
            echo json_encode(["status" => "exists", "message" => "Duplicate"]);
        } else {
            if (move_uploaded_file($_FILES["file"]["tmp_name"], $target_file)) {
                echo json_encode(["status" => "success"]);
            } else {
                echo json_encode(["status" => "error"]);
            }
        }
        break;

    case 'list':
        $files = array_diff(scandir($upload_dir), array('.', '..'));
        $data = [];
        $protocol = (!empty($_SERVER['HTTPS']) && $_SERVER['HTTPS'] !== 'off') ? "https://" : "http://";
        $base_url = $protocol . $_SERVER['HTTP_HOST'] . dirname($_SERVER['PHP_SELF']) . "/" . $upload_dir;
        foreach ($files as $file) {
            $data[] = [
                "name" => $file,
                "url" => $base_url . $file,
                "size" => round(filesize($upload_dir . $file) / 1024, 2) . " KB"
            ];
        }
        echo json_encode(["status" => "success", "data" => array_values($data)]);
        break;

    case 'stats':
        $files = array_diff(scandir($upload_dir), array('.', '..'));
        $total_size = 0;
        foreach ($files as $file) { $total_size += filesize($upload_dir . $file); }
        echo json_encode([
            "status" => "success",
            "total_files" => count($files),
            "storage_used" => round($total_size / (1024 * 1024), 2) . " MB"
        ]);
        break;

    case 'delete':
        $file_name = $_POST['file_name'];
        if (unlink($upload_dir . $file_name)) {
            echo json_encode(["status" => "success"]);
        }
        break;
}
?>
