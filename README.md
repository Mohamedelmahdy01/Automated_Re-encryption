# Automated Re-encryption of SealedSecrets using Kubeseal CLI

**Note**: I’m currently learning Go and may not be proficient in writing complex Go code. This document simplifies the implementation plan to make it more approachable for someone with basic Go knowledge. Additionally, it’s worth noting that the SealedSecrets controller typically retains older private keys, allowing it to decrypt existing SealedSecrets encrypted with those keys. If the key rotation is triggered due to a security concern (e.g., a compromised key), it’s strongly recommended to rotate the underlying Secrets themselves, as simply re-encrypting with a new key may not fully address the issue.

## Overview

This document outlines a simplified plan to add a feature to the `kubeseal` CLI for automatically re-encrypting all SealedSecrets in a Kubernetes cluster. SealedSecrets, managed by Bitnami’s SealedSecrets controller, encrypt Kubernetes Secrets so they can be safely stored in Git or other insecure locations. The controller rotates its key pair every 30 days by default, requiring manual re-encryption of SealedSecrets with the new public key. This feature automates that process, ensuring all SealedSecrets use the latest public key while keeping the implementation simple and secure.

## Approach

The approach uses the fact that the SealedSecrets controller decrypts SealedSecrets into standard Kubernetes Secret objects in the cluster. Since the private key is inaccessible, the CLI will:
1. Fetch the decrypted Secret data.
2. Re-encrypt it with the latest public key using `kubeseal`’s existing encryption logic.
3. Update the SealedSecret objects.

This avoids handling private keys and leverages existing `kubeseal` functionality for simplicity.

Below are the simplified steps with basic Go code snippets integrated into each step. The code assumes minimal Go expertise and focuses on clarity over complexity.

---

## Steps to Implement the Feature

### 1. Identify All Existing SealedSecrets
- **What**: Find all SealedSecret objects in the cluster.
- **How**: Use the Kubernetes API to list SealedSecrets (custom resources: `sealedsecrets.bitnami.com/v1alpha1`). For simplicity, query all namespaces.
- **Why**: We need to know which SealedSecrets to re-encrypt.

**Code Example** (basic Go to list SealedSecrets):
```go
package main

import (
	"context"
	"fmt"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/rest"
	"k8s.io/client-go/tools/clientcmd"
	"github.com/bitnami-labs/sealed-secrets/pkg/apis/sealedsecrets/v1alpha1"
	"k8s.io/client-go/kubernetes/scheme"
)

func listSealedSecrets(config *rest.Config) ([]v1alpha1.SealedSecret, error) {
	// Add SealedSecrets scheme
	v1alpha1.AddToScheme(scheme.Scheme)

	// Create REST client
	client, err := rest.RESTClientFor(config)
	if err != nil {
		return nil, fmt.Errorf("cannot create client: %v", err)
	}

	// Get all SealedSecrets
	result := &v1alpha1.SealedSecretList{}
	err = client.Get().
		Resource("sealedsecrets").
		AbsPath("/apis/sealedsecrets.bitnami.com/v1alpha1").
		Do(context.TODO()).
		Into(result)
	if err != nil {
		return nil, fmt.Errorf("cannot list SealedSecrets: %v", err)
	}

	return result.Items, nil
}

// Example usage
func main() {
	config, err := clientcmd.BuildConfigFromFlags("", "~/.kube/config")
	if err != nil {
		fmt.Printf("Error loading kubeconfig: %v\n", err)
		return
	}

	sealedSecrets, err := listSealedSecrets(config)
	if err != nil {
		fmt.Printf("Error: %v\n", err)
		return
	}

	for _, ss := range sealedSecrets {
		fmt.Printf("Found SealedSecret: %s/%s\n", ss.Namespace, ss.Name)
	}
}
```

**Explanation**: This code connects to the cluster using a kubeconfig file, lists all SealedSecrets, and prints their names and namespaces. It’s simple and uses the `client-go` library, which `kubeseal` already depends on.

---

### 2. Fetch the Latest Public Key
- **What**: Get the controller’s latest public key.
- **How**: Use `kubeseal`’s existing function to fetch the public key. We only need the latest key for re-encryption.
- **Why**: The new key will be used to encrypt the Secret data.

**Code Example** (simplified placeholder):
```go
func getPublicKey(config *rest.Config) ([]byte, error) {
	// Placeholder for kubeseal's built-in function
	publicKey, err := kubesealFetchPublicKey(config) // Assume this exists in kubeseal
	if err != nil {
		return nil, fmt.Errorf("cannot get public key: %v", err)
	}
	return publicKey, nil
}
```

**Explanation**: This assumes `kubeseal` has a function (e.g., `kubesealFetchPublicKey`) to fetch the public key. We don’t need to write this logic since it’s already in `kubeseal`.

---

### 3. Process Each SealedSecret and Update It
- **What**: For each SealedSecret, get its Secret, re-encrypt the data, and update the SealedSecret.
- **How**:
  - Get the Secret with the same name and namespace.
  - If the Secret is missing, skip with a warning.
  - Use the Secret’s data to create a temporary Secret.
  - Encrypt it with `kubeseal`’s logic and the new public key.
  - Update the SealedSecret’s `spec.encryptedData`.
- **Why**: This re-encrypts the data securely and updates the cluster.

**Code Example** (combined processing and updating):
```go
import (
	"context"
	"fmt"
	"k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/kubernetes"
)

func processSealedSecret(clientset *kubernetes.Clientset, ss *v1alpha1.SealedSecret, publicKey []byte) error {
	// Get namespace and name
	namespace := ss.Namespace
	name := ss.Name

	// Get the matching Secret
	secret, err := clientset.CoreV1().Secrets(namespace).Get(context.TODO(), name, metav1.GetOptions{})
	if err != nil {
		fmt.Printf("Warning: Secret %s/%s not found, skipping\n", namespace, name)
		return nil // Skip if Secret is missing
	}

	// Create a temporary Secret with the same data
	tempSecret := &v1.Secret{
		ObjectMeta: metav1.ObjectMeta{
			Name:      name,
			Namespace: namespace,
		},
		Data: secret.Data, // Copy the Secret's data
	}

	// Encrypt the temporary Secret (using kubeseal's logic)
	newSealedSecret, err := encryptSecret(tempSecret, publicKey)
	if err != nil {
		return fmt.Errorf("cannot encrypt %s/%s: %v", namespace, name, err)
	}

	// Update the SealedSecret with new encrypted data
	patchData := []byte(fmt.Sprintf(`{"spec":{"encryptedData":%q}}`, newSealedSecret.Spec.EncryptedData))
	_, err = clientset.CoreV1().RESTClient().
		Patch(types.MergePatchType).
		AbsPath(fmt.Sprintf("/apis/sealedsecrets.bitnami.com/v1alpha1/namespaces/%s/sealedsecrets/%s", namespace, name)).
		Body(patchData).
		Do(context.TODO()).
		Get()
	if err != nil {
		return fmt.Errorf("cannot update %s/%s: %v", namespace, name, err)
	}

	fmt.Printf("Updated SealedSecret %s/%s\n", namespace, name)
	return nil
}

// Placeholder for encryption (to be replaced with kubeseal's real function)
func encryptSecret(secret *v1.Secret, publicKey []byte) (*v1alpha1.SealedSecret, error) {
	// This would use kubeseal's encryption logic
	return &v1alpha1.SealedSecret{
		Spec: v1alpha1.SealedSecretSpec{
			EncryptedData: map[string]string{"example": "encrypted-data"}, // Placeholder
		},
	}, nil
}
```

**Explanation**: This code:
- Fetches the Secret matching the SealedSecret.
- Skips if the Secret is missing (e.g., deleted).
- Copies the Secret’s data into a temporary Secret.
- Calls a placeholder `encryptSecret` function (to be replaced with `kubeseal`’s actual encryption logic).
- Updates the SealedSecret using a patch operation, keeping other fields unchanged.

---

### 4. Add Logging (Bonus)
- **What**: Log the process to track what happens.
- **How**: Print messages for each SealedSecret processed or skipped, with a `--verbose` flag for details.
- **Why**: Helps users understand the process and debug issues.

**Code Example** (integrated into main loop):
```go
var verbose bool // Set via CLI flag, e.g., --verbose

func main() {
	config, err := clientcmd.BuildConfigFromFlags("", "~/.kube/config")
	if err != nil {
		fmt.Printf("Error loading kubeconfig: %v\n", err)
		return
	}

	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		fmt.Printf("Error creating client: %v\n", err)
		return
	}

	publicKey, err := getPublicKey(config)
	if err != nil {
		fmt.Printf("Error getting public key: %v\n", err)
		return
	}

	sealedSecrets, err := listSealedSecrets(config)
	if err != nil {
		fmt.Printf("Error listing SealedSecrets: %v\n", err)
		return
	}

	for _, ss := range sealedSecrets {
		if verbose {
			fmt.Printf("Processing %s/%s...\n", ss.Namespace, ss.Name)
		}
		err := processSealedSecret(clientset, &ss, publicKey)
		if err != nil {
			fmt.Printf("Error processing %s/%s: %v\n", ss.Namespace, ss.Name, err)
		}
	}
}
```

**Explanation**: Adds a `verbose` flag to print detailed messages. Errors are logged but don’t stop the process.

---

### 5. Handle Large Numbers Efficiently (Bonus, Simplified)
- **What**: Process SealedSecrets without overloading the cluster.
- **How**: Stick to sequential processing for simplicity. For large clusters, consider batching later.
- **Why**: Sequential is easier to debug and works for most cases.

**Code Example**: Already handled in the `main` loop above, which processes one SealedSecret at a time.

**Explanation**: The loop in `main` is sequential, avoiding complexity. For advanced users, parallel processing with goroutines could be added later, but it’s not needed for a basic implementation.

---

### 6. Ensure Security (Bonus)
- **What**: Keep private keys secure.
- **How**: The code never accesses private keys, using only the public key and decrypted Secret data.
- **Why**: Maintains the SealedSecrets security model.

---

