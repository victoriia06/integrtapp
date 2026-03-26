## ✅ Исправленный код для всех форм

### 1. **MainForm.cs**
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

        private void buttonTovar_Click(object sender, EventArgs e)
        {
            ProductForm productform = new ProductForm(con);
            productform.ShowDialog();
        }
    }
}
```

---

### 2. **ProductForm.cs**
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

        public ProductForm(NpgsqlConnection connection)
        {
            InitializeComponent();
            con = connection;
            this.StartPosition = FormStartPosition.CenterScreen;
            Update();
        }

        public void Update()
        {
            string sql = "SELECT * FROM Product ORDER BY id";
            NpgsqlDataAdapter da = new NpgsqlDataAdapter(sql, con);
            DataTable dt = new DataTable();
            da.Fill(dt);
            dataGridView1.DataSource = dt;

            if (dataGridView1.Columns.Count >= 3)
            {
                dataGridView1.Columns[0].HeaderText = "ID";
                dataGridView1.Columns[1].HeaderText = "Наименование";
                dataGridView1.Columns[2].HeaderText = "Ед. измерения";
            }
        }

        private void buttonAdd_Click(object sender, EventArgs e)
        {
            AddForm f = new AddForm(con, -1);
            f.ShowDialog();
            Update();
        }

        private void buttonChange_Click(object sender, EventArgs e)
        {
            if (dataGridView1.CurrentRow == null) return;

            int id = (int)dataGridView1.CurrentRow.Cells["id"].Value;
            string name = (string)dataGridView1.CurrentRow.Cells["name"].Value;
            string ed = (string)dataGridView1.CurrentRow.Cells["ed"].Value;

            AddForm f = new AddForm(con, id, name, ed);
            f.ShowDialog();
            Update();
        }

        private void buttonDelete_Click(object sender, EventArgs e)
        {
            if (dataGridView1.CurrentRow == null) return;

            int id = (int)dataGridView1.CurrentRow.Cells["id"].Value;

            DialogResult result = MessageBox.Show("Удалить запись?", "Подтверждение", 
                MessageBoxButtons.YesNo, MessageBoxIcon.Question);

            if (result == DialogResult.Yes)
            {
                NpgsqlCommand command = new NpgsqlCommand("DELETE FROM Product WHERE id = :id", con);
                command.Parameters.AddWithValue("id", id);
                command.ExecuteNonQuery();
                Update();
            }
        }
    }
}
```

---

### 3. **AddForm.cs**
```csharp
using System;
using System.Windows.Forms;
using Npgsql;

namespace Tovar
{
    public partial class AddForm : Form
    {
        NpgsqlConnection con;
        int id;

        public AddForm(NpgsqlConnection con, int id)
        {
            InitializeComponent();
            this.con = con;
            this.id = id;
        }

        public AddForm(NpgsqlConnection con, int id, string nameP, string ed)
        {
            InitializeComponent();
            this.con = con;
            this.id = id;
            textBoxName.Text = nameP;
            textBoxEd.Text = ed;
        }

        private void buttonYes_Click(object sender, EventArgs e)
        {
            if (id == -1)
            {
                NpgsqlCommand command = new NpgsqlCommand("INSERT INTO Product(name, ed) VALUES(:name, :ed)", con);
                command.Parameters.AddWithValue("name", textBoxName.Text);
                command.Parameters.AddWithValue("ed", textBoxEd.Text);
                command.ExecuteNonQuery();
            }
            else
            {
                NpgsqlCommand command = new NpgsqlCommand("UPDATE Product SET name = :name, ed = :ed WHERE id = :id", con);
                command.Parameters.AddWithValue("id", id);
                command.Parameters.AddWithValue("name", textBoxName.Text);
                command.Parameters.AddWithValue("ed", textBoxEd.Text);
                command.ExecuteNonQuery();
            }

            Close();
        }

        private void buttonNo_Click(object sender, EventArgs e)
        {
            Close();
        }
    }
}
```

---

## 📌 Важные моменты:

1. **В MainForm** исправлена опечатка: `Port=5432` (было `Post`), `UserID=postgres` (было `postgers`)

2. **В ProductForm** добавлены три кнопки:
   - `buttonAdd` - добавление
   - `buttonChange` - изменение
   - `buttonDelete` - удаление

3. **В AddForm**:
   - Конструктор с 2 параметрами (`con`, `id = -1`) - для добавления
   - Конструктор с 4 параметрами (`con`, `id`, `nameP`, `ed`) - для изменения

4. **Убраны все `try-catch`** - как вы просили, без вывода ошибок

5. **Убраны `DialogResult`** - форма просто закрывается

Теперь все должно работать без ошибок CS1729!
