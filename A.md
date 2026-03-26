Спасибо за задание! Теперь я полностью понял структуру. У вас в задании используется **MenuStrip** (меню), но вы упростили до кнопок. Это нормально, главное чтобы работало.

Давайте напишем **полностью рабочий код** по вашему заданию, с кнопками вместо меню.

---

## ✅ Полный исправленный код (по заданию)

### 1. **MainForm.cs** (главная форма с кнопками)

```csharp
using System;
using System.Windows.Forms;
using Npgsql;

namespace Tovar
{
    public partial class MainForm : Form
    {
        public NpgsqlConnection con;

        public MainForm()
        {
            InitializeComponent();
        }

        private void MainForm_Load(object sender, EventArgs e)
        {
            this.StartPosition = FormStartPosition.CenterScreen;
            con = new NpgsqlConnection("Server=localhost;Port=5432;UserID=postgres;Password=postpass;Database=MyBase");
            con.Open();
        }

        // Кнопка "Товары"
        private void buttonTovar_Click(object sender, EventArgs e)
        {
            ProductForm fp = new ProductForm(con);
            fp.ShowDialog();
        }

        // Кнопка "Клиенты" (заготовка)
        private void buttonClients_Click(object sender, EventArgs e)
        {
            MessageBox.Show("Форма клиентов будет добавлена позже");
        }

        // Кнопка "Накладные" (заготовка)
        private void buttonInvoices_Click(object sender, EventArgs e)
        {
            MessageBox.Show("Форма накладных будет добавлена позже");
        }

        // Кнопка "Отчеты" (заготовка)
        private void buttonReports_Click(object sender, EventArgs e)
        {
            MessageBox.Show("Форма отчетов будет добавлена позже");
        }
    }
}
```

---

### 2. **ProductForm.cs** (форма с товарами, DataGridView + кнопки)

```csharp
using System;
using System.Data;
using System.Windows.Forms;
using Npgsql;

namespace Tovar
{
    public partial class ProductForm : Form
    {
        public NpgsqlConnection con;
        DataTable dt = new DataTable();
        DataSet ds = new DataSet();

        public ProductForm(NpgsqlConnection connection)
        {
            InitializeComponent();
            con = connection;
            this.StartPosition = FormStartPosition.CenterScreen;
            Update();
        }

        public void Update()
        {
            string sql = "SELECT * FROM Product";
            NpgsqlDataAdapter da = new NpgsqlDataAdapter(sql, con);
            ds.Clear();
            da.Fill(ds);
            dt = ds.Tables[0];
            dataGridView1.DataSource = dt;

            // Настройка заголовков
            if (dataGridView1.Columns.Count >= 3)
            {
                dataGridView1.Columns[0].HeaderText = "Номер";
                dataGridView1.Columns[1].HeaderText = "Наименование";
                dataGridView1.Columns[2].HeaderText = "Ед.измерения";
            }
        }

        // Кнопка "Добавить"
        private void buttonAdd_Click(object sender, EventArgs e)
        {
            AddProductForm f = new AddProductForm(con, -1);
            f.ShowDialog();
            Update();
        }

        // Кнопка "Изменить"
        private void buttonChange_Click(object sender, EventArgs e)
        {
            if (dataGridView1.CurrentRow == null)
            {
                MessageBox.Show("Выберите запись для изменения");
                return;
            }

            int id = (int)dataGridView1.CurrentRow.Cells["ID"].Value;
            string name = (string)dataGridView1.CurrentRow.Cells["name"].Value;
            string ed = (string)dataGridView1.CurrentRow.Cells["ed"].Value;

            AddProductForm f = new AddProductForm(con, id, name, ed);
            f.ShowDialog();
            Update();
        }

        // Кнопка "Удалить"
        private void buttonDelete_Click(object sender, EventArgs e)
        {
            if (dataGridView1.CurrentRow == null)
            {
                MessageBox.Show("Выберите запись для удаления");
                return;
            }

            int id = (int)dataGridView1.CurrentRow.Cells["ID"].Value;

            DialogResult result = MessageBox.Show("Удалить запись?", "Подтверждение", 
                MessageBoxButtons.YesNo, MessageBoxIcon.Question);

            if (result == DialogResult.Yes)
            {
                NpgsqlCommand command = new NpgsqlCommand("DELETE FROM Product WHERE ID = :id", con);
                command.Parameters.AddWithValue("id", id);
                command.ExecuteNonQuery();
                Update();
            }
        }

        // Кнопка "Выход"
        private void buttonExit_Click(object sender, EventArgs e)
        {
            this.Close();
        }
    }
}
```

---

### 3. **AddProductForm.cs** (форма добавления/изменения)

```csharp
using System;
using System.Windows.Forms;
using Npgsql;

namespace Tovar
{
    public partial class AddProductForm : Form
    {
        NpgsqlConnection con;
        int id;

        // Конструктор для ДОБАВЛЕНИЯ (id = -1)
        public AddProductForm(NpgsqlConnection con, int id)
        {
            InitializeComponent();
            this.con = con;
            this.id = id;
            this.Text = "Добавление товара";
        }

        // Конструктор для ИЗМЕНЕНИЯ (передаем id, name, ed)
        public AddProductForm(NpgsqlConnection con, int id, string nameP, string ed)
        {
            InitializeComponent();
            this.con = con;
            this.id = id;
            textBoxName.Text = nameP;
            textBoxEd.Text = ed;
            this.Text = "Изменение товара";
        }

        // Кнопка "Подтвердить"
        private void buttonYes_Click(object sender, EventArgs e)
        {
            if (id == -1) // Добавление
            {
                NpgsqlCommand command = new NpgsqlCommand("INSERT INTO Product (name, ed) VALUES (:name, :ed)", con);
                command.Parameters.AddWithValue("name", textBoxName.Text);
                command.Parameters.AddWithValue("ed", textBoxEd.Text);
                command.ExecuteNonQuery();
            }
            else // Изменение
            {
                NpgsqlCommand command = new NpgsqlCommand("UPDATE Product SET name = :name, ed = :ed WHERE ID = :id", con);
                command.Parameters.AddWithValue("id", id);
                command.Parameters.AddWithValue("name", textBoxName.Text);
                command.Parameters.AddWithValue("ed", textBoxEd.Text);
                command.ExecuteNonQuery();
            }

            Close();
        }

        // Кнопка "Отмена"
        private void buttonNo_Click(object sender, EventArgs e)
        {
            Close();
        }
    }
}
```

---

### 4. **Program.cs**

```csharp
using System;
using System.Windows.Forms;

namespace Tovar
{
    internal static class Program
    {
        [STAThread]
        static void Main()
        {
            Application.EnableVisualStyles();
            Application.SetCompatibleTextRenderingDefault(false);
            Application.Run(new MainForm());
        }
    }
}
```

---

## 📋 Что нужно на формах (дизайн)

### MainForm:
- Кнопки: `buttonTovar`, `buttonClients`, `buttonInvoices`, `buttonReports`

### ProductForm:
- `dataGridView1` - таблица
- Кнопки: `buttonAdd`, `buttonChange`, `buttonDelete`, `buttonExit`

### AddProductForm:
- `textBoxName` - поле для наименования
- `textBoxEd` - поле для единицы измерения
- `buttonYes` - кнопка подтверждения
- `buttonNo` - кнопка отмены

---

## ✅ Теперь все должно работать:
1. Запускается `MainForm`
2. Подключается к БД
3. Нажимаете "Товары" → открывается `ProductForm` с таблицей
4. Кнопки "Добавить", "Изменить", "Удалить" работают
5. Форма `AddProductForm` используется и для добавления (id=-1), и для изменения (id>0)
