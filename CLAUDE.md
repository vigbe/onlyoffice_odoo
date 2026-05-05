# CLAUDE.md - Guía de Desarrollo para Odoo 19

> **ARCHIVO DE MEMORIA CLAUDE** - Documentación técnica para asistentes AI
> 
> Esta documentación contiene las mejores prácticas, arquitectura estándar y reglas de desarrollo para módulos de Odoo 19.

---

## Información Fundamental de Odoo 19

### Cambios Clave en Odoo 19 (desde Odoo 18)

| Cambio | Antes | Ahora |
|--------|-------|-------|
| List view tag | `<tree>` | `<list>` |
| Atributos dinámicos | `attrs="{'invisible': [...]}"` | `invisible="..."` |
| Validación de delete | Override `unlink()` | `@api.ondelete(at_uninstall=False)` |
| Agregación de campos | `group_operator=` | `aggregator=` |
| Queries SQL | String concatenation | `odoo.tools.SQL` class |

### Framework Base
- **Python 3.10+**: Versión mínima requerida
- **OWL Framework**: JavaScript completamente basado en OWL
- **ORM mejorado**: Mejor performance y nuevas funcionalidades
- **Multicompany**: Soporte nativo con `_check_company_auto = True`

---

## Estructura Estándar de Módulos

```
mi_modulo/
├── __init__.py                    # Obligatorio
├── __manifest__.py                # Obligatorio
├── controllers/
│   ├── __init__.py
│   └── main.py                    # [a-z0-9_]
├── data/
│   ├── data.xml                   # Datos maestros
│   ├── demo.xml                   # Datos de demostración
│   └── cron.xml                   # Tareas programadas
├── models/
│   ├── __init__.py
│   └── my_model.py                # Modelos
├── security/
│   ├── ir.model.access.csv        # Obligatorio - permisos
│   └── security_groups.xml        # Grupos de usuarios
├── static/
│   ├── description/
│   │   └── icon.png
│   └── src/
│       ├── js/                    # JavaScript OWL
│       ├── scss/                  # SCSS (4 espacios)
│       └── xml/                   # Templates QWeb
├── views/
│   └── my_model_views.xml         # Vistas
├── wizard/
│   ├── __init__.py
│   └── my_wizard.py               # TransientModel
├── tests/
│   ├── __init__.py
│   └── test_my_model.py
└── migrations/
    └── 19.0.1.0/
        └── post-migration.py
```

---

## Archivo Manifest (__manifest__.py)

### Estructura Completa

```python
{
    'name': 'Nombre del Módulo',
    'version': '19.0.1.0.0',                # Formato Odoo 19
    'category': 'Tools',
    'summary': 'Descripción corta',
    'description': """
        Descripción detallada del módulo.
    """,
    'author': 'Tu Empresa',
    'website': 'https://www.tuempresa.com',
    'license': 'LGPL-3',
    'depends': [
        'base',                              # Siempre requerido
        'mail',                              # Para chatter
        # Dependencias específicas
    ],
    'data': [
        # ORDEN CRÍTICO - Seguridad primero
        'security/security_groups.xml',
        'security/ir.model.access.csv',
        # Datos
        'data/data.xml',
        # Vistas
        'views/my_model_views.xml',
    ],
    'demo': [
        'data/demo.xml',
    ],
    'assets': {
        'web.assets_backend': [
            'mi_modulo/static/src/js/backend.js',
            'mi_modulo/static/src/scss/backend.scss',
        ],
    },
    'installable': True,
    'application': False,
    'auto_install': False,
    'external_dependencies': {
        'python': [],
        'bin': [],
    },
}
```

### Hooks (opcional)

```python
# En __init__.py
def pre_init_hook(env):
    """Ejecutado antes de instalar"""
    pass

def post_init_hook(env):
    """Ejecutado después de instalar"""
    pass

# En __manifest__.py
'pre_init_hook': 'pre_init_hook',
'post_init_hook': 'post_init_hook',
```

---

## Definición de Modelos

### Estructura Base

```python
import logging
from odoo import models, fields, api, _
from odoo.exceptions import ValidationError, UserError

_logger = logging.getLogger(__name__)


class MiModelo(models.Model):
    _name = 'mi.modelo'
    _description = 'Descripción del Modelo'
    _inherit = ['mail.thread', 'mail.activity.mixin']
    _order = 'name desc'
    _rec_name = 'name'
    _check_company_auto = True

    # ===== CAMPOS BÁSICOS =====
    name = fields.Char(
        string='Nombre',
        required=True,
        tracking=True,
    )
    
    active = fields.Boolean(default=True)
    
    company_id = fields.Many2one(
        'res.company',
        string='Empresa',
        required=True,
        default=lambda self: self.env.company
    )
    
    # ===== SELECCIÓN =====
    state = fields.Selection([
        ('draft', 'Borrador'),
        ('confirmed', 'Confirmado'),
        ('done', 'Realizado'),
    ], default='draft', tracking=True)
    
    # ===== RELACIONALES =====
    partner_id = fields.Many2one(
        'res.partner',
        string='Cliente',
        domain="['|', ('company_id', '=', False), ('company_id', '=', company_id)]",
        check_company=True
    )
    
    line_ids = fields.One2many(
        'mi.modelo.line',
        'parent_id',
        string='Líneas',
        copy=True
    )
    
    tag_ids = fields.Many2many('mi.tag', string='Etiquetas')
    
    # ===== MONETARIOS =====
    amount = fields.Monetary(
        string='Monto',
        currency_field='currency_id',
    )
    
    currency_id = fields.Many2one(
        'res.currency',
        default=lambda self: self.env.company.currency_id
    )
    
    # ===== COMPUTADOS =====
    total = fields.Monetary(
        compute='_compute_total',
        store=True,
        currency_field='currency_id'
    )
    
    @api.depends('line_ids.amount')
    def _compute_total(self):
        for record in self:
            record.total = sum(record.line_ids.mapped('amount'))
    
    # ===== CONSTRAINTS =====
    @api.constrains('amount')
    def _check_positive_amount(self):
        for record in self:
            if record.amount < 0:
                raise ValidationError(_('El monto debe ser positivo'))
    
    # ===== ONDELETE (Odoo 18+) =====
    @api.ondelete(at_uninstall=False)
    def _unlink_if_not_draft(self):
        if any(rec.state != 'draft' for rec in self):
            raise UserError(_('No se pueden eliminar registros en estado no borrador'))
    
    # ===== ONCHANGE =====
    @api.onchange('partner_id')
    def _onchange_partner_id(self):
        if self.partner_id:
            self.currency_id = self.partner_id.currency_id
    
    # ===== CRUD =====
    @api.model_create_multi
    def create(self, vals_list):
        for vals in vals_list:
            if not vals.get('name'):
                vals['name'] = self.env['ir.sequence'].next_by_code('mi.modelo')
        return super().create(vals_list)
    
    # ===== ACCIONES =====
    def action_confirm(self):
        self.ensure_one()
        if self.state != 'draft':
            raise UserError(_('Solo se pueden confirmar borradores'))
        self.state = 'confirmed'
```

### Orden de Elementos en Modelos

1. Atributos privados (`_name`, `_description`, `_inherit`, etc.)
2. `default_get` y métodos default
3. Declaración de campos (agrupados por tipo)
4. Métodos compute, inverse, search
5. Métodos `@api.constrains`
6. Métodos `@api.ondelete`
7. Métodos `@api.onchange`
8. Métodos CRUD (`create`, `write`, `unlink`)
9. Métodos de acción (`action_*`)
10. Otros métodos de negocio

---

## Vistas en Odoo 19

### Vista List (NO usar `<tree>`)

```xml
<record id="mi_modelo_view_list" model="ir.ui.view">
    <field name="name">mi.modelo.list</field>
    <field name="model">mi.modelo</field>
    <field name="arch" type="xml">
        <list string="Mi Modelo" editable="bottom" multi_edit="1">
            <field name="name" required="1"/>
            <field name="partner_id"/>
            <field name="state" widget="badge" 
                   decoration-success="state == 'done'"
                   decoration-warning="state == 'confirmed'"
                   decoration-danger="state == 'cancelled'"/>
            <field name="total" sum="Total"/>
        </list>
    </field>
</record>
```

### Vista Form

```xml
<record id="mi_modelo_view_form" model="ir.ui.view">
    <field name="name">mi.modelo.form</field>
    <field name="model">mi.modelo</field>
    <field name="arch" type="xml">
        <form string="Mi Modelo">
            <header>
                <button name="action_confirm" string="Confirmar" 
                        type="object" class="btn-primary"
                        invisible="state != 'draft'"/>
                <field name="state" widget="statusbar" 
                       statusbar_visible="draft,confirmed,done"/>
            </header>
            <sheet>
                <group>
                    <group>
                        <field name="name"/>
                        <field name="partner_id"/>
                    </group>
                    <group>
                        <field name="currency_id"/>
                        <field name="total"/>
                    </group>
                </group>
                <notebook>
                    <page string="Líneas">
                        <field name="line_ids">
                            <list editable="bottom">
                                <field name="name"/>
                                <field name="amount"/>
                            </list>
                        </field>
                    </page>
                </notebook>
            </sheet>
            <div class="oe_chatter">
                <field name="message_follower_ids"/>
                <field name="activity_ids"/>
                <field name="message_ids"/>
            </div>
        </form>
    </field>
</record>
```

### Cambios Críticos en Vistas Odoo 19

| Odoo 17- | Odoo 18+ |
|----------|----------|
| `<tree>` | `<list>` |
| `attrs="{'invisible': [('state', '=', 'done')]}"` | `invisible="state == 'done'"` |
| `attrs="{'readonly': [('state', '=', 'done')]}"` | `readonly="state == 'done'"` |
| `attrs="{'required': [('type', '=', 'sale')]}"` | `required="type == 'sale'"` |

---

## Seguridad

### Grupos de Usuarios

```xml
<odoo>
    <record id="module_category_mi_modulo" model="ir.module.category">
        <field name="name">Mi Módulo</field>
        <field name="sequence">50</field>
    </record>

    <record id="group_mi_modulo_user" model="res.groups">
        <field name="name">Usuario</field>
        <field name="category_id" ref="module_category_mi_modulo"/>
        <field name="implied_ids" eval="[(4, ref('base.group_user'))]"/>
    </record>

    <record id="group_mi_modulo_manager" model="res.groups">
        <field name="name">Administrador</field>
        <field name="category_id" ref="module_category_mi_modulo"/>
        <field name="implied_ids" eval="[(4, ref('group_mi_modulo_user'))]"/>
    </record>
</odoo>
```

### Derechos de Acceso (CSV)

```csv
id,name,model_id:id,group_id:id,perm_read,perm_write,perm_create,perm_unlink
access_mi_modelo_user,mi.modelo.user,model_mi_modelo,group_mi_modulo_user,1,1,1,0
access_mi_modelo_manager,mi.modelo.manager,model_mi_modelo,group_mi_modulo_manager,1,1,1,1
```

### Reglas de Registro (Multi-empresa)

```xml
<odoo>
    <record id="mi_modelo_company_rule" model="ir.rule">
        <field name="name">Mi Modelo: multi-empresa</field>
        <field name="model_id" ref="model_mi_modelo"/>
        <field name="domain_force">['|', ('company_id', '=', False), ('company_id', 'in', company_ids)]</field>
    </record>
</odoo>
```

---

## Decoradores @api

### @api.depends (campos computados)

```python
@api.depends('price', 'qty', 'partner_id.email')
def _compute_total(self):
    for record in self:
        record.total = record.price * record.qty
```

### @api.constrains (validación)

```python
@api.constrains('date_start', 'date_end')
def _check_dates(self):
    for record in self:
        if record.date_end < record.date_start:
            raise ValidationError(_('La fecha fin debe ser posterior'))
```

### @api.ondelete (validación de eliminación - Odoo 18+)

```python
@api.ondelete(at_uninstall=False)
def _unlink_if_not_draft(self):
    if any(rec.state != 'draft' for rec in self):
        raise UserError(_('No se puede eliminar'))
```

### @api.onchange (UI)

```python
@api.onchange('partner_id')
def _onchange_partner_id(self):
    if self.partner_id:
        self.currency_id = self.partner_id.currency_id
```

### @api.model_create_multi (batch create)

```python
@api.model_create_multi
def create(self, vals_list):
    for vals in vals_list:
        vals.setdefault('name', 'Nuevo')
    return super().create(vals_list)
```

---

## Campos Comandos (One2many/Many2many)

```python
from odoo.fields import Command

# Crear
'line_ids': [Command.create({'name': 'Test'})]

# Actualizar
'line_ids': [Command.update(line_id, {'qty': 10})]

# Eliminar
'line_ids': [Command.delete(line_id)]

# Limpiar todo
'line_ids': [Command.clear()]

# Reemplazar con IDs
'line_ids': [Command.set([id1, id2])]

# Agregar sin borrar existentes
'tag_ids': [Command.link(tag_id)]

# Desvincular
'tag_ids': [Command.unlink(tag_id)]
```

---

## JavaScript OWL

### Componente Estándar

```javascript
/** @odoo-module **/

import { Component, useState, onWillStart } from "@odoo/owl";
import { registry } from "@web/core/registry";
import { useService } from "@web/core/utils/hooks";

export class MiComponente extends Component {
    static template = "mi_modulo.MiComponente";
    static props = {
        record: Object,
    };

    setup() {
        this.orm = useService("orm");
        this.notification = useService("notification");
        this.state = useState({ loading: false });

        onWillStart(async () => {
            await this.loadData();
        });
    }

    async loadData() {
        this.state.loading = true;
        try {
            this.state.data = await this.orm.call("mi.modelo", "get_data", []);
        } finally {
            this.state.loading = false;
        }
    }
}

registry.category("fields").add("mi_widget", {
    component: MiComponente,
});
```

---

## Reglas Críticas para AI

### DEBE HACER

1. **SIEMPRE** usar `<list>` en lugar de `<tree>`
2. **SIEMPRE** usar atributos directos (`invisible`, `readonly`, `required`)
3. **SIEMPRE** incluir `ir.model.access.csv`
4. **SIEMPRE** usar `store=True` en campos computados frecuentemente consultados
5. **SIEMPRE** usar `@api.depends()` explícito
6. **SIEMPRE** iterar `for record in self:` en métodos computados
7. **SIEMPRE** usar `self.ensure_one()` en métodos de acción
8. **SIEMPRE** usar `@api.ondelete` en lugar de override `unlink()`
9. **SIEMPRE** usar `@api.model_create_multi` para `create()`
10. **SIEMPRE** usar `from odoo.fields import Command` para operaciones relacionales

### NO DEBE HACER

1. **NUNCA** usar `attrs` (deprecated)
2. **NUNCA** usar `<tree>` (usar `<list>`)
3. **NUNCA** llamar `cr.commit()` manualmente
4. **NUNCA** usar jQuery (solo OWL)
5. **NUNCA** hardcodear valores configurables
6. **NUNCA** usar SQL directo cuando ORM es suficiente
7. **NUNCA** usar `eval()` - usar `ast.literal_eval()`

---

## Convenciones de Nomenclatura

| Elemento | Convención |
|----------|------------|
| Clases Python | CamelCase (`AccountInvoice`) |
| Variables de modelo | camelCase |
| Variables comunes | snake_case |
| Many2One | sufijo `_id` |
| One2Many/Many2many | sufijo `_ids` |
| Métodos compute | `_compute_<field>` |
| Métodos acción | `action_*` |
| XML IDs vistas | `<model>_view_<type>` |
| XML IDs acciones | `<model>_action` |
| XML IDs menús | `<model>_menu` |

---

## Referencias

- **Skill Odoo 19**: `.agents/skills/odoo-19/SKILL.md`
- **Guías detalladas**: `.agents/skills/odoo-19/dev/`
- **Odoo 19 Docs**: https://github.com/odoo/documentation/tree/19.0

---

*Documentación actualizada para Odoo 19 - ONLYOFFICE Odoo*
