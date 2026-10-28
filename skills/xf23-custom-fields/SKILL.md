---
name: xf23-custom-fields
description: Complete vendor-neutral guide and workflow for implementing XenForo 2.3 Custom Fields on any content entity (6-layer architecture, schema, entities, repository, container listener, controllers, and templates).
---

# XenForo 2.3 Custom Fields Architecture Guide

This skill provides a vendor-neutral, step-by-step reference for implementing native XenForo 2.3 Custom Fields on any custom content entity (`{Vendor}\{Addon}:{Entity}`).

---

## Architecture Overview (6 Layers)

Adding Custom Fields to a content entity `{Vendor}\{Addon}:{Entity}` requires 6 coordinated layers:

```
1. Database Tables  --> xf_{vendor}_{addon}_{type}_field
                    --> xf_{vendor}_{addon}_{type}_field_value
                    --> xf_{vendor}_{addon}_{type} (column: custom_fields)

2. Entities         --> Entity/{Type}Field.php (extends XF\Entity\AbstractField)
                    --> Entity/{Type}FieldValue.php (extends XF\Mvc\Entity\Entity)
                    --> Entity/{Type}.php (uses XF:CustomFieldsHolder behavior)

3. Repository       --> Repository/{Type}Field.php (extends XF\Repository\AbstractField)

4. Container Sync   --> Listener.php (app_setup -> customFields.{registryKey})

5. Admin Controller --> Admin/Controller/{Type}Field.php (extends XF\Admin\Controller\AbstractField)
                    --> Admin/Controller/{Type}.php (bulkSet custom fields on save)

6. Templates        --> Admin Field List / Edit / Delete templates
                    --> Content Edit Template (<xf:macro id="public:custom_fields_macros::custom_fields_edit" ... />)
```

---

## Step-by-Step Implementation Guide

### 1. Database Schema (`Setup.php`)

Create 2 new tables (`_field` and `_field_value`) and add `custom_fields` to the content entity table (and ensure all 3 are reverted in `uninstallStep*`):

```php
// 1. Definition Table
$tables['xf_vendor_addon_type_field'] = function (Create $table) {
    $table->addColumn('field_id', 'varbinary', 25);
    $table->addColumn('display_order', 'int')->setDefault(1);
    $table->addColumn('field_type', 'varbinary', 25)->setDefault('textbox');
    $table->addColumn('field_choices', 'blob');
    $table->addColumn('match_type', 'varbinary', 25)->setDefault('none');
    $table->addColumn('match_params', 'blob');
    $table->addColumn('max_length', 'int')->setDefault(0);
    $table->addColumn('required', 'tinyint')->setDefault(0);
    $table->addColumn('display_template', 'text')->setDefault('');
    $table->addColumn('wrapper_template', 'text')->setDefault('');
    $table->addColumn('display_group', 'varbinary', 25)->setDefault('above_info');

    $table->addPrimaryKey('field_id');
    $table->addKey(['display_group', 'display_order'], 'display_group_order');
};

// 2. Value Table
$tables['xf_vendor_addon_type_field_value'] = function (Create $table) {
    $table->addColumn('content_id', 'int')->unsigned(true);
    $table->addColumn('field_id', 'varbinary', 25);
    $table->addColumn('field_value', 'mediumtext');

    $table->addPrimaryKey(['content_id', 'field_id']);
    $table->addKey('field_id');
};

// 3. Content Table (MUST have custom_fields column for XF:CustomFieldsHolder)
$table->addColumn('custom_fields', 'mediumblob')->nullable();
```

---

### 2. Field Definition Entity (`Entity/{Type}Field.php`)

```php
<?php

declare(strict_types=1);

namespace Vendor\Addon\Entity;

use XF\Entity\AbstractField;
use XF\Mvc\Entity\Structure;

class TypeField extends AbstractField
{
	protected function getClassIdentifier()
	{
		return 'Vendor\Addon:TypeField';
	}

	protected static function getPhrasePrefix()
	{
		return 'vendor_addon_type_field';
	}

	protected function _postDelete()
	{
		$this->db()->delete('xf_vendor_addon_type_field_value', 'field_id = ?', $this->field_id);
		parent::_postDelete();
	}

	public static function getStructure(Structure $structure)
	{
		self::setupDefaultStructure(
			$structure,
			'xf_vendor_addon_type_field',
			'Vendor\Addon:TypeField',
			[
				'groups' => [
					'above_info', 'below_info', 'sidebar', 'extra_tab', 'new_tab',
				],
				'has_wrapper_template' => true,
			]
		);

		return $structure;
	}
}
```

> **CRITICAL**: Always pass `'has_wrapper_template' => true` in `setupDefaultStructure` options when `wrapper_template` exists in the DB table. Otherwise MySQL throws error `1364 (Field 'wrapper_template' doesn't have a default value)`.

---

### 3. Field Value Entity (`Entity/{Type}FieldValue.php`)

```php
<?php

declare(strict_types=1);

namespace Vendor\Addon\Entity;

use XF\Mvc\Entity\Entity;
use XF\Mvc\Entity\Structure;

class TypeFieldValue extends Entity
{
	public static function getStructure(Structure $structure)
	{
		$structure->table = 'xf_vendor_addon_type_field_value';
		$structure->shortName = 'Vendor\Addon:TypeFieldValue';
		$structure->primaryKey = ['content_id', 'field_id'];
		$structure->columns = [
			'content_id' => ['type' => self::UINT, 'required' => true],
			'field_id' => ['type' => self::STR, 'maxLength' => 25, 'required' => true],
			'field_value' => ['type' => self::STR, 'default' => ''],
		];
		$structure->relations = [
			'Field' => [
				'entity' => 'Vendor\Addon:TypeField',
				'type' => self::TO_ONE,
				'conditions' => 'field_id',
				'primary' => true,
			],
		];

		return $structure;
	}
}
```

---

### 4. Content Entity (`Entity/{Type}.php`)

```php
<?php

declare(strict_types=1);

namespace Vendor\Addon\Entity;

use XF\CustomField\Set;
use XF\Mvc\Entity\Entity;
use XF\Mvc\Entity\Structure;

class Type extends Entity
{
	public function getCustomFields(): Set
	{
		$class = Set::class;
		$class = $this->app()->extendClass($class);

		$fieldDefinitions = $this->app()->container('customFields.vendorAddonTypeFields');

		return new $class($fieldDefinitions, $this);
	}

	public static function getStructure(Structure $structure)
	{
		// ... standard structure setup ...
		$structure->columns['custom_fields'] = ['type' => self::JSON_ARRAY, 'default' => []];

		$structure->behaviors['XF:CustomFieldsHolder'] = [
			'valueTable' => 'xf_vendor_addon_type_field_value',
			'keyColumn' => 'content_id',
		];

		$structure->getters['custom_fields'] = true;

		$structure->relations['CustomFields'] = [
			'entity' => 'Vendor\Addon:TypeFieldValue',
			'type' => self::TO_MANY,
			'conditions' => 'content_id',
			'key' => 'field_id',
		];

		return $structure;
	}
}
```

---

### 5. Field Repository (`Repository/{Type}Field.php`)

```php
<?php

declare(strict_types=1);

namespace Vendor\Addon\Repository;

use XF\Repository\AbstractField;

class TypeField extends AbstractField
{
	protected function getRegistryKey()
	{
		return 'vendorAddonTypeFields';
	}

	protected function getClassIdentifier()
	{
		return 'Vendor\Addon:TypeField';
	}

	public function getDisplayGroups()
	{
		return [
			'above_info' => \XF::phrase('vendor_addon_above_info'),
			'below_info' => \XF::phrase('vendor_addon_below_info'),
			'sidebar' => \XF::phrase('vendor_addon_sidebar'),
			'extra_tab' => \XF::phrase('vendor_addon_extra_tab'),
			'new_tab' => \XF::phrase('vendor_addon_own_tab'),
		];
	}
}
```

---

### 6. Code Event Listener (`Listener.php` & `app_setup`)

Register the `customFields.<registryKey>` container factory in `Listener::appSetup`:

```php
public static function appSetup(\XF\App $app): void
{
	$container = $app->container();

	$container['customFields.vendorAddonTypeFields'] = $app->fromRegistry(
		'vendorAddonTypeFields',
		function (\XF\Container $c) {
			return $c['em']->getRepository('Vendor\Addon:TypeField')->rebuildFieldCache();
		},
		function (array $fieldInfo) use ($app) {
			$class = \XF\CustomField\DefinitionSet::class;
			$class = $app->extendClass($class);

			return new $class($fieldInfo);
		}
	);
}
```

---

### 7. Admin Controllers

#### Field Controller (`Admin/Controller/{Type}Field.php`)
```php
<?php

declare(strict_types=1);

namespace Vendor\Addon\Admin\Controller;

use XF\Admin\Controller\AbstractField;
use XF\Mvc\ParameterBag;

class TypeField extends AbstractField
{
	protected function preDispatchController($action, ParameterBag $params)
	{
		$this->assertAdminPermission('vendorAddonPermission');
	}

	protected function getClassIdentifier()
	{
		return 'Vendor\Addon:TypeField';
	}

	protected function getLinkPrefix()
	{
		return 'vendor-addon/custom-fields';
	}

	protected function getTemplatePrefix()
	{
		return 'vendor_addon_type_field';
	}
}
```

#### Content Controller (`Admin/Controller/{Type}.php`)
In `saveProcess()`:
```php
$customFields = $this->filter('custom_fields', 'array');
$customFieldsShown = $this->filter('custom_fields_shown', 'array-str');
$form->setup(function () use ($entity, $customFields, $customFieldsShown) {
	/** @var \XF\CustomField\Set $fieldSet */
	$fieldSet = $entity->custom_fields;
	$fieldSet->bulkSet($customFields, $customFieldsShown, 'admin', true);
});
```

---

### 8. Templates

#### Admin Edit Form (`vendor_addon_type_edit.html`)
> **CRITICAL**: Use `public:` prefix and `arg-editMode="admin"` (camelCase attribute, NO hyphens in macro arg names):
> [!IMPORTANT]
> In XenForo templates, always pass `arg-set="{$entity.getCustomFields()}"` to `<xf:macro id="public:custom_fields_macros::custom_fields_edit" ... />`. Passing `arg-set="{$entity.custom_fields}"` directly can evaluate to the raw column array instead of the `\XF\CustomField\Set` object if `$structure->columns` contains `custom_fields`.

```html
<h3 class="block-formSectionHeader">{{ phrase('vendor_addon_custom_fields') }}</h3>

<xf:macro id="public:custom_fields_macros::custom_fields_edit"
	arg-type="vendorAddonTypeFields"
	arg-set="{$entity.getCustomFields()}"
	arg-editMode="admin" />
```

---

## Pitfalls & Best Practices Summary

| Issue / Pitfall | Cause | Correct Fix |
|---|---|---|
| **MySQL 1364 (`wrapper_template` has no default)** | `AbstractField::setupDefaultStructure` omits `wrapper_template` from structure columns if option is missing. | Pass `['has_wrapper_template' => true]` in `setupDefaultStructure` options. |
| **`Unknown column custom_fields` error on save** | `XF:CustomFieldsHolder` behavior calls `$entity->getValue('custom_fields')` which requires a real column definition. | Add `'custom_fields'` JSON column to database table and entity structure. |
| **Macro argument syntax error (`arg-edit-mode`)** | XenForo template parser requires macro argument names to contain only `A-Z, 0-9, _`. | Use camelCase `arg-editMode="admin"`. Never use hyphens in macro `arg-*` names. |
| **`Template admin:custom_fields_macros is unknown`** | `custom_fields_macros` is in `public` template space, not `admin`. | Prefix macro ID with `public:` -> `public:custom_fields_macros::custom_fields_edit`. |
| **Custom fields hidden in AdminCP form** | Macro `editMode` defaults to `'user'`, filtering out admin-only fields. | Always pass `arg-editMode="admin"` in AdminCP edit forms. |
