# All Bundled Framework Recipes

Deployer ships one recipe per framework. Require it in `deploy.php`
(`require 'recipe/<name>.php';`); each is based on the `common` recipe and
follows the same override-and-hook pattern described in the skill.

- CakePHP — `recipe/cakephp.php`
- CodeIgniter — `recipe/codeigniter.php`
- CodeIgniter 4 — `recipe/codeigniter4.php`
- Composer — `recipe/composer.php`
- Contao — `recipe/contao.php`
- Craft CMS — `recipe/craftcms.php`
- Drupal 7 — `recipe/drupal7.php`
- Drupal 8 — `recipe/drupal8.php`
- Flow Framework — `recipe/flow_framework.php`
- FuelPHP — `recipe/fuelphp.php`
- Joomla — `recipe/joomla.php`
- Laravel — `recipe/laravel.php`
- Magento — `recipe/magento.php`
- Magento 2 — `recipe/magento2.php`
- Pimcore — `recipe/pimcore.php`
- PrestaShop — `recipe/prestashop.php`
- Shopware — `recipe/shopware.php`
- SilverStripe — `recipe/silverstripe.php`
- Spiral — `recipe/spiral.php`
- Statamic — `recipe/statamic.php`
- Sulu — `recipe/sulu.php`
- Symfony — `recipe/symfony.php`
- TYPO3 — `recipe/typo3.php`
- WordPress — `recipe/wordpress.php`
- Yii2 — `recipe/yii.php`
- Zend Framework — `recipe/zend_framework.php`

There is also a `common` recipe (`recipe/common.php`) that all of the above
build on, and a `provision` recipe (`recipe/provision.php`) for server setup.
