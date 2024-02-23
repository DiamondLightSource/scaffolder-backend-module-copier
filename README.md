# scaffolder-backend-module-copier

Welcome to the `fetch:copier` action for the `scaffolder-backend`.

## Getting started

You need to configure the action in your backend:

## From your Backstage root directory

```bash
# From your Backstage root directory
yarn add --cwd packages/backend https://github.com/DiamondLightSource/scaffolder-backend-module-copier.git
```

Configure the action within `packages/backend/src/plugins/scaffolder.ts`:
(you can check the [docs](https://backstage.io/docs/features/software-templates/writing-custom-actions#registering-custom-actions) to see all options):

```typescript
import { CatalogClient } from '@backstage/catalog-client';
import { ScmIntegrations } from '@backstage/integration';
import {
  createBuiltinActions,
  createRouter,
} from '@backstage/plugin-scaffolder-backend';
import { createFetchCopierAction } from '@backstage/plugin-scaffolder-backend-module-copier';
import { Router } from 'express';
import type { PluginEnvironment } from '../types';

export default async function createPlugin(
  env: PluginEnvironment,
): Promise<Router> {
  const catalogClient = new CatalogClient({
    discoveryApi: env.discovery,
  });
  const integrations = ScmIntegrations.fromConfig(env.config);
  
  const builtInActions = createBuiltinActions({
    integrations,
    catalogClient,
    config: env.config,
    reader: env.reader,
  });
  const actions = [
    ...builtInActions, 
    createFetchCopierAction({ integrations, reader: env.reader })
  ];
  
  return await createRouter({
    catalogClient,
    actions,
    logger: env.logger,
    config: env.config,
    database: env.database,
    reader: env.reader,
    identity: env.identity,
    scheduler: env.scheduler,
  });
}
```

After that you can use the action in your template:

```yaml
apiVersion: scaffolder.backstage.io/v1beta3
kind: Template
metadata:
  name: copier-demo
  title: Copier Test
  description: Copier example
spec:
  owner: backstage/techdocs-core
  type: service

  parameters:
    - title: Fill in some steps
      required:
        - name
        - owner
      properties:
        name:
          title: Name
          type: string
          description: Unique name of the component
          ui:autofocus: true
          ui:options:
            rows: 5
        owner:
          title: Owner
          type: string
          description: Owner of the component
          ui:field: OwnerPicker
          ui:options:
            allowedKinds:
              - Group
        system:
          title: System
          type: string
          description: System of the component
          ui:field: EntityPicker
          ui:options:
            allowedKinds:
              - System
            defaultKind: System

    - title: Choose a location
      required:
        - repoUrl
        - dryRun
      properties:
        repoUrl:
          title: Repository Location
          type: string
          ui:field: RepoUrlPicker
          ui:options:
            allowedHosts:
              - github.com
        dryRun:
          title: Only perform a dry run, don't publish anything
          type: boolean
          default: false

  steps:
    - id: fetch-base
      name: Fetch Base
      action: fetch:copier
      input:
        url: "<complete url to template repo>"
        values:
          name: ${{ parameters.name }}
          owner: ${{ parameters.owner }}
          system: ${{ parameters.system }}
          destination: ${{ parameters.repoUrl | parseRepoUrl }}

    - id: publish
      if: ${{ parameters.dryRun !== true }}
      name: Publish
      action: publish:github
      input:
        allowedHosts:
          - github.com
        description: This is ${{ parameters.name }}
        repoUrl: ${{ parameters.repoUrl }}

    - name: Results
      if: ${{ parameters.dryRun }}
      action: debug:log
      input:
        listWorkspace: true

  output:
    links:
      - title: Repository
        url: ${{ steps.publish.output.remoteUrl }}
```

You can also visit the `/create/actions` route in your Backstage application to find out more about the parameters this action accepts when it's installed to configure how you like.

### Environment setup

While running Backstage from a Docker container, you need to have `copier` installed within your image.

You can do so by including the following lines in the last step of your Dockerfile:

```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends python3-pip
RUN pip install --break-system-packages copier
```
