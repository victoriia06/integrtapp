using System;
using System.Collections.Generic;
using System.ComponentModel;
using System.Data;
using System.Drawing;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
using System.Windows.Forms;
using Npgsql;

namespace Tovar
{
    public partial class ProductForm : Form
    {
        public NpgsqlConnection con;
        public ProductForm()
        {
            InitializeComponent();
        }

        private void buttonChange_Click(object sender, EventArgs e)
        {
            int id = (int)dataGridView1.CurrentRow.Cells["ID"].Value;
            string name = (string)dataGridView1.CurrentRow.Cells["name"].Value;
            string ed = (string)dataGridView1.CurrentRow.Cells["ed"].Value;
            AddForm f = new AddForm(con, id, name, ed);
            f.ShowDialog();
            Update();
        }
    }
}


using System;
using System.Collections.Generic;
using System.ComponentModel;
using System.Data;
using System.Drawing;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
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
            con = new NpgsqlConnection("Server=localhost;Post=5432;UserID=postgers;Password=postpass;Database=MyBase");
            con.Open();
        }

        private void buttonTovar_Click(object sender, EventArgs e)
        {
            try
            {
                ProductForm productform = new ProductForm(con);
                productform.ShowDialog();
            }
            catch { }
        }


    }
}


using System;
using System.Collections.Generic;
using System.ComponentModel;
using System.Data;
using System.Drawing;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
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
            textBoxEd.Text = ed;
            textBoxName.Text = nameP;
        }
        private void buttonYes_Click(object sender, EventArgs e)
        {
            if (id == -1)
            {
                try
                {
                    NpgsqlCommand command = new NpgsqlCommand("insert into Product(name,ed) values(:name,:ed)", con);
                    command.Parameters.AddWithValue("name", textBoxName.Text);
                    command.Parameters.AddWithValue("ed", textBoxEd.Text);
                    command.ExecuteNonQuery();
                    Close();
                }
                catch { }
            }
            else
            {
                try
                {
                    NpgsqlCommand command = new NpgsqlCommand("update product set name=:name, ed=:ed where id=:id", con);
                    command.Parameters.AddWithValue("id", id);
                    command.Parameters.AddWithValue("name", textBoxName.Text);
                    command.Parameters.AddWithValue("ed", textBoxEd.Text);
                    command.ExecuteNonQuery();
                    Close();
                }
                catch { }
            }

        }

        private void buttonNo_Click(object sender, EventArgs e)
        {
            Close();
        }

    }
}



