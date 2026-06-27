## Project and processes

https://learn.microsoft.com/en-us/dynamics365/guidance/business-processes/about-configure-azure-devops-project-processes

Check if packages are installed:

```python
python -c "import importlib.util; packages=['pandas','requests','openpyxl']; print('\n'.join(f'{name}: installed' if importlib.util.find_spec(name) else f'{name}: not installed' for name in packages))"
```

### Personal access token

https://dev.azure.com/fhinway/_usersSettings/tokens
BPCMAR26

- Organization: does not exist?
- Project and Team: Read, write, & manage
- Work Items: Read, write, & manage
- Process and Work Item Types: does not exist?

Process and Work Item Types scope is probably part of Work Items.

**June Preview**

- Extensions: Read & manage
- Marketplace: Aquire

Read only permissions are not sufficient.

## Page layout

https://learn.microsoft.com/en-us/dynamics365/guidance/business-processes/about-configure-azure-devops-page-layout

In Excel, copy column "Field name" in sheet "Fields" to column "Label".

## June Preview

setup_wizard.py takes parameters from environment variables if not specified as command line arguments. 

multiple changes in the import logic, mostly the catalog import (see commits)
