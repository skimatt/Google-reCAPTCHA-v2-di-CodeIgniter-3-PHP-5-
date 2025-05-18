
# Tutorial Integrasi Google reCAPTCHA v2 di CodeIgniter 3 (PHP 5)

Tutorial ini akan memandu Anda langkah demi langkah untuk menambahkan Google reCAPTCHA v2 ke halaman login di CodeIgniter 3 menggunakan PHP 5.

---

## Prasyarat

- Proyek CodeIgniter 3 yang sudah berjalan dengan PHP 5.
- Akses ke Google reCAPTCHA (untuk mendapatkan site key dan secret key).
- Basic pemahaman CI3 MVC (Controller, View, Model).

---

## 1. Daftar dan Dapatkan reCAPTCHA Keys

1. Kunjungi [Google reCAPTCHA Admin Console](https://www.google.com/recaptcha/admin/create).
2. Pilih tipe reCAPTCHA: **reCAPTCHA v2** > "I'm not a robot" Checkbox.
3. Masukkan label, domain (contoh: `localhost` untuk testing lokal).
4. Setujui Terms dan klik **Submit**.
5. Anda akan mendapatkan:
   - **Site Key** (untuk form frontend)
   - **Secret Key** (untuk verifikasi backend)

---

## 2. Tambahkan reCAPTCHA ke View Login Anda

Buka file view login Anda, misalnya di `application/views/auth/login.php`.

Tambahkan kode berikut tepat sebelum tombol submit:

```html
<div class="mb-4">
  <div class="recaptcha-wrapper">
    <div class="g-recaptcha" data-sitekey="YOUR_SITE_KEY"></div>
  </div>
  <?php if (form_error('g-recaptcha-response')): ?>
    <p class="text-danger text-sm mt-1"><?= form_error('g-recaptcha-response'); ?></p>
  <?php endif; ?>
</div>

<style>
  .recaptcha-wrapper {
    transform: scale(0.85);
    transform-origin: 0 0;
    max-width: 100%;
    overflow: hidden;
  }

  @media (min-width: 400px) {
    .recaptcha-wrapper {
      transform: scale(0.95);
    }
  }

  @media (min-width: 576px) {
    .recaptcha-wrapper {
      transform: scale(1);
    }
  }
</style>

<script src="https://www.google.com/recaptcha/api.js" async defer></script>
````

> Ganti `YOUR_SITE_KEY` dengan site key yang Anda dapatkan dari Google.

---

## 3. Modifikasi Controller Login Anda

Buka controller login Anda, misalnya di `application/controllers/Auth.php`.

Tambahkan validasi dan verifikasi reCAPTCHA seperti ini:

```php
public function login() {
    // Jika sudah login, redirect sesuai role
    if ($this->session->userdata('email')) {
        $role = $this->session->userdata('role');
        switch ($role) {
            case 'admin': redirect('admin/dashboard'); break;
            case 'dokter': redirect('dokter/dashboard'); break;
            case 'bidan': redirect('bidan/dashboard'); break;
            case 'pasien': redirect('pasien/dashboard'); break;
            default:
                log_message('error', 'Unknown role in session: ' . $role);
                $this->session->set_flashdata('error', 'Role tidak dikenal. Silakan login ulang.');
                redirect('auth/login');
        }
    }

    $data['title'] = 'Login';

    // Validasi form email dan password
    $this->form_validation->set_rules('email', 'Email', 'required|valid_email');
    $this->form_validation->set_rules('password', 'Password', 'required');
    // Validasi reCAPTCHA, jangan lupa nama input-nya 'g-recaptcha-response'
    $this->form_validation->set_rules('g-recaptcha-response', 'Captcha', 'required', [
        'required' => 'Silakan verifikasi CAPTCHA terlebih dahulu.'
    ]);

    if ($this->form_validation->run() === FALSE) {
        $this->load->view('auth/login', $data);
    } else {
        // Verifikasi reCAPTCHA
        $recaptcha_response = $this->input->post('g-recaptcha-response');
        $secret_key = 'YOUR_SECRET_KEY'; // Ganti dengan secret key Anda
        $verify = file_get_contents("https://www.google.com/recaptcha/api/siteverify?secret={$secret_key}&response={$recaptcha_response}");
        $response_data = json_decode($verify);

        if (!$response_data->success) {
            $this->session->set_flashdata('error', 'Verifikasi CAPTCHA gagal. Silakan coba lagi.');
            $this->load->view('auth/login', $data);
            return;
        }

        // Proses login biasa
        $email = $this->input->post('email', TRUE);
        $password = $this->input->post('password', TRUE);
        $user = $this->db->get_where('tb_users', ['email' => $email, 'is_active' => 1])->row();

        if ($user && password_verify($password, $user->password)) {
            $session_data = [
                'uuid' => $user->uuid,
                'email' => $user->email,
                'nama' => $user->nama,
                'role' => $user->role
            ];
            $this->session->set_userdata($session_data);
            log_message('info', 'User logged in: ' . $user->email . ' with role: ' . $user->role);

            // Redirect berdasarkan role
            switch ($user->role) {
                case 'admin': redirect('admin/dashboard'); break;
                case 'dokter': redirect('dokter/dashboard'); break;
                case 'bidan': redirect('bidan/dashboard'); break;
                case 'pasien': redirect('pasien/dashboard'); break;
                default:
                    log_message('error', 'Unknown role after login: ' . $user->role);
                    $this->session->set_flashdata('error', 'Role tidak dikenal. Silakan hubungi admin.');
                    redirect('auth/login');
            }
        } else {
            $this->session->set_flashdata('error', 'Email, password salah, atau akun tidak aktif.');
            $this->load->view('auth/login', $data);
        }
    }
}
```

> Ganti `YOUR_SECRET_KEY` dengan secret key dari Google.

---

## 4. Uji Coba

* Buka halaman login (`http://localhost/puskesmas/auth/login`).
* Pastikan reCAPTCHA muncul dengan benar dan responsif.
* Coba submit login tanpa mencentang CAPTCHA, harus muncul error.
* Coba login dengan CAPTCHA valid, pastikan login berjalan sesuai role.

---

## 5. Troubleshooting

* **CAPTCHA tidak muncul?**
  Pastikan script Google reCAPTCHA sudah disisipkan dengan benar di view.

* **Verifikasi CAPTCHA selalu gagal?**
  Periksa kembali `secret_key` dan domain Anda di Google reCAPTCHA admin console.

* **Captcha terpotong di mobile?**
  Pastikan CSS `.recaptcha-wrapper` seperti contoh di atas sudah ada.

---

## Kesimpulan

Dengan mengikuti tutorial ini, Anda berhasil mengamankan form login dengan Google reCAPTCHA v2 di CodeIgniter 3 menggunakan PHP 5, sekaligus memastikan tampilan reCAPTCHA responsif untuk berbagai ukuran layar.

---

## Referensi

* [Google reCAPTCHA Docs](https://developers.google.com/recaptcha/docs/v2)
* [CodeIgniter 3 User Guide](https://codeigniter.com/userguide3/)

---

**Selamat mencoba!**
Jika ada pertanyaan, silakan hubungi saya kembali.
